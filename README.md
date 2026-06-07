# Thuraya GMR-1 Faraday Cage Replay — CONFIRMED WORKING

## A/B Test Results (June , 2026)

**Bias tee OFF (LNA disabled):**
```
0, 0, 0, 0, 2, 0, 0, 0, 0, 0, 0, 0  ← essentially zero crc=0
```

**Bias tee ON (LNA active):**
```
67, 197, 201  ← hundreds of crc=0 frames per cycle
```

---

## Hardware Setup (Faraday Corner)

- **USRP B210** (Zhixun clone) → omni antenna on TX/RX port
- **RTL-SDR Blog v3** → RTL-SDR Blog L-band active patch antenna → **bias tee ON**
- ~30cm air gap between antennas
- Both inside Faraday corner

---

## Source File

ARFCN 267 (BCCH) → 1,533,343,750 Hz
Downloaded from: http://people.osmocom.org/tnt/gmr/data/tnt-locupd-267-93600.cfile.bz2

### Upsample (Mac, conda gnuradio-env):

```bash
cd ~/Desktop
python3 -c "
import numpy as np
from scipy.signal import resample_poly
iq = np.fromfile('tnt-locupd-267-93600.cfile', dtype=np.complex64)
up = resample_poly(iq, 1000000, 93600).astype(np.complex64)
up.tofile('thuraya_tx_1M_Jun_arfcn267.cfile')
print(f'Input:  {len(iq)} samples at 93,600 Hz')
print(f'Output: {len(up)} samples at 1,000,000 Hz')
"
```

Output: 48,572,832 samples = 48.57 seconds per TX loop

---

## Runtime Commands

### Mac — TX (start first)

```bash
cd /usr/local/Cellar/uhd/4.7.0.0/lib/uhd/examples

./tx_samples_from_file \
  --file "$HOME/Desktop/thuraya_tx_1M_Jun_arfcn267.cfile" \
  --rate 1e6 \
  --freq 1533343750 \
  --gain 20 \
  --type float \
  --args "type=b200" \
  --repeat
```

### Ubuntu — Fresh FIFO (before each run)

```bash
rm -f /tmp/arfcn_267.cfile
mkfifo /tmp/arfcn_267.cfile
```

### Ubuntu Terminal 1 — GSMTAP monitor

```bash
sudo tshark -i lo -f 'port 4729' -Y 'gsmtap' -V
```

### Ubuntu Terminal 2 — RX capture (gmr1_rx_sdr.py)

```bash
conda activate gmr-env
cd ~/Desktop/osmo-gmr

python utils/gmr1_rx_sdr.py \
  -s 1024000 \
  -B L \
  -f 1533150000 \
  -a 267 \
  -g 40 \
  --args 'rtl=0,bias=1'
```

### Ubuntu Terminal 3 — Live decode

```bash
cd ~/Desktop/osmo-gmr
./src/gmr1_rx_live 4 267:/tmp/arfcn_267.cfile
```

Real-time decode runs ~40 seconds (one TX file pass).

### Alternative: Loop decode for continuous monitoring

```bash
while true; do ./src/gmr1_rx_live 4 267:/tmp/arfcn_267.cfile 2>&1 | grep -c "crc=0"; sleep 2; done
```

---

## What Success Looks Like

- tshark shows BCCH System Information Type 1 (all segments)
- Paging Request Type 3 with GPS Almanac data
- Satellite Position: 1.9°S, 44.1°E (Thuraya 2)
- Orbital Radius: 42,181 km
- MCC 901, MNC 5, LAC 0x0520, Spot Beam ID 288
- Hundreds of crc=0 frames per decode cycle

---

## Key Findings

1. **Bias tee (LNA) is essential** for over-air — without it, almost zero crc=0
2. **Coax loopback** works at gain 0-5 without LNA (197 crc=0 per run)
3. **gmr1_rx_sdr.py** 2to3 conversion is clean — no math bugs
4. **FIFO pipe** enables real-time decode: gmr1_rx_sdr.py → FIFO → gmr1_rx_live
5. **B210 type arg** must be "type=b200" (not "type=b210")
6. **gmr1_rx_live** provides ~40s of continuous real-time decode per pass

---

## Diagnostic Tests Performed

| Test | Result |
|------|--------|
| Original cfile → gmr1_rx | ✅ crc=0 (baseline works) |
| Upsample → downsample → gmr1_rx | ✅ crc=0 (resampling clean) |
| Coax + rtl_sdr + Python channelize → gmr1_rx | ✅ crc=0 (RF chain works) |
| Coax + gmr1_rx_sdr.py → gmr1_rx | ✅ 197 crc=0 (script works) |
| Faraday + bias tee OFF | ❌ ~0 crc=0 |
| Faraday + bias tee ON | ✅ 67-201 crc=0 per cycle |
