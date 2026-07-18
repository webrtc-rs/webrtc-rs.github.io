## Appendix: reproduce it yourself

Everything above was measured on an AMD Ryzen 7 1700 (8c/16t), Linux, loopback. Here are the exact
steps.

**Prerequisites:** a recent Rust toolchain, Go ≥ 1.24, and [`poop`](https://github.com/andrewrk/poop).
`poop` reads kernel perf counters, so lower the paranoia level first (throughput-only runs don't need
this — only the `poop` counter tables do):

```sh
sudo sysctl -w kernel.perf_event_paranoid=1
```

**webrtc-rs side** — current master, with the in-tree multi-pair harness:

```sh
git clone https://github.com/webrtc-rs/webrtc && cd webrtc
git submodule update --init --recursive
cargo build --release --example data-channels-flow-control-multi
# one connection, unordered/no-retransmit, dedicated reactor on:
FLOW_PAIRS=1 FLOW_WARMUP_MB=16 FLOW_STOP_MB=48 FLOW_ORDERED=0 FLOW_DEDICATED_REACTOR=1 \
  ./target/release/examples/data-channels-flow-control-multi
```

The harness prints a `FINAL <Mbps> aggregate over <N> pairs …` line — that's per-interval
steady-state throughput (the `FLOW_WARMUP_MB` ramp is skipped, then throughput is measured over
`FLOW_STOP_MB`). Env knobs:

| variable | meaning | default |
|---|---|---|
| `FLOW_PAIRS`             | concurrent connection pairs (aggregate) | 10 |
| `FLOW_WARMUP_MB`         | ramp skipped before measuring | 64 |
| `FLOW_STOP_MB`           | bytes measured per pair | 128 |
| `FLOW_ORDERED`           | `1` = ordered/reliable, else unordered/no-rtx | 0 |
| `FLOW_DEDICATED_REACTOR` | `1` = opt-in dedicated reactor thread per PC | 1 |

**Pion side** — pin the exact release, drop in a matching harness, build it:

```sh
git clone https://github.com/pion/webrtc pion-webrtc && cd pion-webrtc
git checkout v4.2.16
mkdir -p examples/flow-control-multi
# save the Go file below as examples/flow-control-multi/main.go
go build -o /tmp/pion-flow-multi ./examples/flow-control-multi
```

The Pion harness mirrors the webrtc-rs one exactly — same env knobs, same 1 KB / 512 KB–1 MB
watermarks, same per-interval steady-state measurement, and crucially a **non-trickle** handshake
(wait for gathering to complete, then exchange the full SDP) so it doesn't race on
`AddICECandidate` under concurrency:

```go
// examples/flow-control-multi/main.go  (SPDX-License-Identifier: MIT)
package main

import (
	"fmt"
	"os"
	"strconv"
	"sync"
	"sync/atomic"
	"time"

	"github.com/pion/webrtc/v4"
)

const (
	bufferedAmountLowThreshold uint64 = 512 * 1024
	maxBufferedAmount          uint64 = 1024 * 1024
	msgSize                           = 1024
)

func check(err error) {
	if err != nil {
		panic(err)
	}
}

func envInt(key string, def int) int {
	if v := os.Getenv(key); v != "" {
		if n, err := strconv.Atoi(v); err == nil {
			return n
		}
	}
	return def
}

func runPair(warmupBytes, stopBytes uint64, ordered bool, results chan<- float64) {
	cfg := webrtc.Configuration{ICEServers: []webrtc.ICEServer{}}
	offerPC, err := webrtc.NewPeerConnection(cfg)
	check(err)
	answerPC, err := webrtc.NewPeerConnection(cfg)
	check(err)

	var stop atomic.Bool

	// Receiver: per-interval steady-state.
	answerPC.OnDataChannel(func(dc *webrtc.DataChannel) {
		var got, measureStartBytes uint64
		var measuring atomic.Bool
		var measureStart time.Time
		dc.OnMessage(func(m webrtc.DataChannelMessage) {
			got += uint64(len(m.Data))
			if !measuring.Load() && got >= warmupBytes {
				measureStart, measureStartBytes = time.Now(), got
				measuring.Store(true)
			}
			if measuring.Load() && got-measureStartBytes >= stopBytes {
				secs := time.Since(measureStart).Seconds()
				mbps := float64((got-measureStartBytes)*8) / secs / (1024.0 * 1024.0)
				if stop.CompareAndSwap(false, true) {
					results <- mbps
				}
			}
		})
	})

	// Sender.
	maxRetransmits := uint16(0)
	orderedFlag := ordered
	opts := &webrtc.DataChannelInit{Ordered: &orderedFlag, MaxRetransmits: &maxRetransmits}
	if ordered {
		opts = &webrtc.DataChannelInit{Ordered: &orderedFlag}
	}
	dc, err := offerPC.CreateDataChannel("data", opts)
	check(err)
	dc.SetBufferedAmountLowThreshold(bufferedAmountLowThreshold)

	sendMore := make(chan struct{}, 1)
	dc.OnBufferedAmountLow(func() {
		select {
		case sendMore <- struct{}{}:
		default:
		}
	})
	dc.OnOpen(func() {
		buf := make([]byte, msgSize)
		for {
			if stop.Load() {
				return
			}
			if err := dc.Send(buf); err != nil {
				return
			}
			if dc.BufferedAmount() > maxBufferedAmount {
				select {
				case <-sendMore:
				case <-time.After(50 * time.Millisecond):
				}
			}
		}
	})

	// Non-trickle handshake: wait for gathering, exchange full SDP.
	offer, err := offerPC.CreateOffer(nil)
	check(err)
	offerGathered := webrtc.GatheringCompletePromise(offerPC)
	check(offerPC.SetLocalDescription(offer))
	<-offerGathered
	check(answerPC.SetRemoteDescription(*offerPC.LocalDescription()))

	answer, err := answerPC.CreateAnswer(nil)
	check(err)
	answerGathered := webrtc.GatheringCompletePromise(answerPC)
	check(answerPC.SetLocalDescription(answer))
	<-answerGathered
	check(offerPC.SetRemoteDescription(*answerPC.LocalDescription()))
}

func main() {
	pairs := envInt("FLOW_PAIRS", 10)
	warmupBytes := uint64(envInt("FLOW_WARMUP_MB", 64)) * 1024 * 1024
	stopBytes := uint64(envInt("FLOW_STOP_MB", 128)) * 1024 * 1024
	ordered := os.Getenv("FLOW_ORDERED") == "1"

	results := make(chan float64, pairs)
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < pairs; i++ {
		wg.Add(1)
		go func() { defer wg.Done(); runPair(warmupBytes, stopBytes, ordered, results) }()
	}
	agg := 0.0
	for i := 0; i < pairs; i++ {
		agg += <-results
	}
	fmt.Printf("FINAL %.3f Mbps aggregate over %d pairs (ordered=%v) in %.3fs\n",
		agg, pairs, ordered, time.Since(start).Seconds())
	os.Exit(0)
}
```

**The A/B itself.** Set matching env for both binaries, read the `FINAL` lines for steady-state
throughput, and hand both to `poop` for the counter table (Benchmark 1 is the baseline):

```sh
export LC_ALL=C \
  FLOW_PAIRS=10 FLOW_WARMUP_MB=16 FLOW_STOP_MB=48 FLOW_ORDERED=0 FLOW_DEDICATED_REACTOR=1

# steady-state throughput (run each a few times, take the median):
/tmp/pion-flow-multi
./target/release/examples/data-channels-flow-control-multi

# full hardware-counter A/B (pion = baseline, webrtc-rs = benchmark 2):
poop /tmp/pion-flow-multi ./target/release/examples/data-channels-flow-control-multi
```

Two gotchas that cost me time: `export LC_ALL=C` if your shell is in a comma-decimal locale (it
corrupts `awk` arithmetic in any wrapper script), and remember that `poop`'s `wall_time` includes
per-connection setup and is latency-bound at N=1 — the `FINAL` steady-state line is the honest
throughput number, the `poop` table is for the per-byte CPU and RSS counters.
