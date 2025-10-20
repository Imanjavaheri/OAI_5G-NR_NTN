# Implementing OAI 5G NR NTN LEO Simulation

This guide walks through configuring and running a Standalone (SA) 5G New Radio Non-Terrestrial Network (NR NTN) Low Earth Orbit (LEO) simulation with OpenAirInterface (OAI) softmodems and the integrated RFsimulator.

## 1. Prerequisites and Setup 🧭

These instructions assume the OAI source tree is already built and located in `~/openairinterface5g/`.

1. **Locate the build directory**
   ```bash
   cd ~/openairinterface5g/cmake_targets/ran_build/build/
   ```
2. **Verify configuration file availability**
   - gNB config: `../../../ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf`
   - UE config: `../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf`

## 2. Launch the gNB (LEO Ground Station) 🛰️

Run the gNB softmodem as the RFsimulator server with the LEO-specific configuration.

```bash
# Terminal 1 (gNB)
sudo -E ./nr-softmodem \
  -O ../../../ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf \
  --rfsim
```

Keep this terminal open after confirming initialization logs (including orbit data).

## 3. Launch the NR UE (User Equipment) 📱

Start the UE softmodem in a separate terminal so it can act as the RFsimulator client and apply the required LEO compensations.

```bash
# Terminal 2 (UE)
sudo -E ./nr-uesoftmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \
  --band 254 -C 2488400000 --CO -873500000 -r 25 --numerology 0 --ssb 60 \
  --rfsim --rfsimulator.prop_delay 20 --rfsimulator.options chanmod \
  --time-sync-I 0.1 \
  --ntn-initial-time-drift -46 \
  --autonomous-ta \
  --initial-fo 57340 \
  --cont-fo-comp 2
```

### Key UE Command-Line Parameters

| Option | Purpose in LEO simulation |
| ------ | ------------------------- |
| `--rfsimulator.prop_delay 20` | Sets the channel model propagation delay (e.g., 20 ms). |
| `--rfsimulator.options chanmod` | Activates the LEO channel model. |
| `--cont-fo-comp 2` | Enables continuous Doppler-focused frequency offset compensation. |
| `--initial-fo 57340` | Applies the known initial Doppler frequency offset for synchronization. |
| `--ntn-initial-time-drift -46` | Configures the initial downlink time drift (µs/s) before SIB19 reception. |
| `--autonomous-ta` | Allows autonomous Timing Advance updates driven by downlink drift. |

With both processes running, the UE should synchronize to the gNB using LEO-specific compensations.

