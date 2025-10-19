# 5G NR NTN LEO Simulation with Netcat (noS1 Mode)

This repository documents the procedure for validating bidirectional user-plane connectivity over a simulated 5G Non-Terrestrial Network (NTN) Low Earth Orbit (LEO) link using the OpenAirInterface (OAI) RF simulator. The workflow leverages the lightweight `netcat` (`nc`) utility to exchange custom text messages between a gNB and a UE operating in **noS1** mode.

## Prerequisites

- OAI source tree checked out under `~/openairinterface5g/`
- Build artifacts located at `~/openairinterface5g/cmake_targets/ran_build/build/`
- Ability to run commands with `sudo`
- High-efficiency execution flags enabled: `-E` and `--parallel-config PARALLEL_SINGLE_THREAD`

## Step-by-Step Procedure

### 1. Navigate to the Build Directory

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build/
```

### 2. Launch the gNB (Terminal 1)

Start the gNB with RF simulator support and **noS1** mode enabled. This creates the `oaitun_enb1` tunnel interface (IP address `10.0.1.1`).

```bash
sudo ./nr-softmodem \
  -O ../../../ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf \
  --rfsim \
  --rfsimulator.options chanmod \
  --noS1 \
  -E \
  --parallel-config PARALLEL_SINGLE_THREAD
```

### 3. Launch the NR UE (Terminal 2)

In a separate terminal, start the NR UE. This creates the `oaitun_ue1` tunnel interface (IP address `10.0.1.2`) and applies Doppler and timing compensation suitable for an NTN LEO scenario.

```bash
sudo ./nr-uesoftmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \
  --band 254 \
  -C 2488400000 \
  --CO -873500000 \
  -r 25 \
  --numerology 0 \
  --ssb 60 \
  --rfsim \
  --rfsimulator.prop_delay 20 \
  --rfsimulator.options chanmod \
  --time-sync-I 0.1 \
  --ntn-initial-time-drift -46 \
  --autonomous-ta \
  --initial-fo 57340 \
  --cont-fo-comp 2 \
  --noS1 \
  -E \
  --parallel-config PARALLEL_SINGLE_THREAD
```

Monitor both terminals for the RRC Connection Setup messages to confirm that the UE has successfully attached to the gNB.

### 4. Downlink Messaging (gNB → UE)

#### a. UE Listener (Terminal 4)

Open a listener on the UE tunnel interface (`10.0.1.2`) using port `6000`.

```bash
nc -l 10.0.1.2 6000
```

#### b. gNB Sender (Terminal 3)

Connect from the gNB tunnel interface to the UE listener and send a message.

```bash
nc 10.0.1.2 6000
```

Type a message, for example:

```
Hello LEO, message from gNB!
```

The message should immediately appear in the UE listener terminal. Use `Ctrl+C` in both terminals to end the session.

### 5. Uplink Messaging (UE → gNB)

#### a. gNB Listener (Terminal 3)

```bash
nc -l 10.0.1.1 6001
```

#### b. UE Sender (Terminal 4)

```bash
nc 10.0.1.1 6001
```

Send a reply such as:

```
Reply received! Sending uplink data from UE.
```

The message should appear in the gNB terminal. Press `Ctrl+C` in both terminals to close the session.

## Outcome

Completing the steps above validates bidirectional custom text transmission over the simulated 5G NTN LEO link in noS1 mode using OAI's RF simulator and the `netcat` utility.
