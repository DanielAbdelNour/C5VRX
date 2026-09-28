# Direct Gain V3 core, issue #107

The experimental build enables Direct Gain V3 for the default automatic gain
profile. Golden video output remains the existing baseline; this branch changes
only the RF gain actuator. The ordinary Direct Gain V2 observer still supplies
fusion diagnostics but never writes gain while V3 is compiled in.

V3 reads four separated 64-byte regions from the most recently completed RX
descriptor every 1 ms. A 256-entry lookup converts raw Q4/I4 bytes to centered
power, near-origin occupancy and ADC rail occupancy. The observer computes
P50, P95 and a Phase8-code coherence trust signal. It checks that the sampled
descriptor remains completed throughout the copy and does not pace RX DMA or
the BitScrambler video path.

The current experimental hold region is centered P50 13..32, P95 <=65,
clip <20 per thousand, origin <=250 per thousand, and coherence >=55%.
A healthy signal makes zero gain writes. A weak coherent signal may gain up;
poor coherence with adequate amplitude is not treated as a request for more
gain. A saturated Q4 envelope takes a bounded emergency physical step down,
without pretending its clipped power can be inverted exactly.

The controller decodes the ESP32-C5 vendor table into RF stage, BB bank and
Fine state. It first evaluates known or locally inferred Fine states in the
current bank, then learned BB or RF destinations. Unknown bank responses are
not assigned fabricated dB values. If no measured destination is available,
it makes one structural Fine, BB, or RF range move and verifies the outcome.
Clean before/after observations update a global relative power-gain model for
the receiver's own tuples. Unstable pre-write input or an implausible Fine
response is rejected. Each tuple also records measured settle time, an IQ
quality-change proxy for artifacts, and repeated bad-state evidence. The
proxy cannot prove whether a sub-millisecond video flash was visible. The
model never maps an absolute RSSI or antenna type to a fixed gain index.

After a physical write, V3 ignores samples too close to the write and requires
three fresh, stable envelope observations before verification. The observed
settle time is recorded separately for Fine, BB and RF changes. This is an
observer-resolution measure until the issue #105 gain-step ring oracle proves
the actual analog transition time. A small residual may use one Fine fix.

The thresholds and local Fine prior still need live hardware validation.
The first required A/B is to confirm zero writes on stationary clean video,
then move close/far and compare visible static, Q4 headroom, write counts and
settle response with Direct Gain V2. Do not promote V3 to main based on host
tests or a firmware build alone.
