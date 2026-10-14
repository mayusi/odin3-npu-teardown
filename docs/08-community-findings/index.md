# Community Findings

Work contributed by people outside the original investigation. Each entry is credited to its author and linked to the original report.

## 🏆 [The HTP runs under Linux](htp-runs-under-linux.md) — by [@WhiteEagle-12](https://github.com/WhiteEagle-12)

!!! success "This overturned the main verdict of this repo"
    Running Linux on an Odin 3 with the identical CQ8725S SoC, **@WhiteEagle-12** booted the CDSP, loaded the QNN HTP V79 backend, and ran real neural-network graphs on it — with output matching the CPU reference to a correlation of **0.9999997**, at roughly **4.5 TOPS INT8**.

    The original 47-phase investigation was done entirely from the Android side and concluded the NPU was fused off in silicon. This result shows the silicon is present and working; stock Android simply never starts it.

    It is the single most important finding in this repository. [Read the full write-up →](htp-runs-under-linux.md) · [Original report (issue #1)](https://github.com/mayusi/odin3-npu-teardown/issues/1)

---

Have a data point from your own device? Open an issue — see [CONTRIBUTING.md](https://github.com/mayusi/odin3-npu-teardown/blob/main/CONTRIBUTING.md).
