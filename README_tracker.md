🛰 5G NTN Real-Time Simulation and Telemetry Visualization

This project uses OpenAirInterface (OAI) in RF simulator (rfsim) mode to emulate a Non-Terrestrial Network (NTN) link between a gNB and a UE.
A Python script reads the live log from the gNB, plots orbit and link telemetry in real time, and records everything in CSV.

⚙️ Requirements
sudo apt update
sudo apt install python3 python3-pip python3-tk python3-pyqt5 -y
pip3 install matplotlib numpy

🧩 Run sequence
1️⃣ gNB

Run from your OAI build folder:

cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo -E ./nr-softmodem   \
  -O ../../../ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf   \
  --rfsim | tee -a ~/oai_ntn.log

2️⃣ UE

Run in another terminal:

cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo -E ./nr-uesoftmodem   \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf   \
  --band 254 -C 2488400000 --CO -873500000 -r 25 --numerology 0 --ssb 60   \
  --rfsim --rfsimulator.prop_delay 20 --rfsimulator.options chanmod   \
  --time-sync-I 0.1 --ntn-initial-time-drift -46 --autonomous-ta   \
  --initial-fo 57340 --cont-fo-comp 2

3️⃣ Tracker

When the gNB is printing [HW] Satellite orbit lines, start the tracker:

python3 ~/ntn_sat_tracker.py

📊 Output

Live window showing

3-D satellite orbit

Orbital phase

Delay

Doppler shift

Azimuth φ and Elevation θ

~/ntn_data.csv

Time(s),x,y,z,vx,vy,vz,Lat,Lon,Alt,Phi,Theta,Phase,Delay,Doppler,Vel_toward_gNB,Acc_toward_gNB
1.7307e+09,0.0,-1452983.84,6824948.81,0.0,7392.32,1573.77,67.05,-90.00,606900,-91.20,66.04,92.3,10.14,35.56,6602.65,3.98

🧠 Notes
Issue	Fix
CSV empty	Ensure gNB uses tee -a ~/oai_ntn.log
Blank plot	Run tracker while gNB is active
GUI error	sudo apt install python3-pyqt5 -y
Font warning	Safe to ignore
📜 ntn_sat_tracker.py

Save this script in your home folder as ~/ntn_sat_tracker.py.

import re, math, time, os, csv
import matplotlib
matplotlib.use('Qt5Agg')
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D
import numpy as np

LOG_FILE = "/home/iotpolimi/oai_ntn.log"
CSV_FILE = "/home/iotpolimi/ntn_data.csv"

def eci_to_geo(x, y, z):
    R_earth = 6371000.0
    r = math.sqrt(x**2 + y**2 + z**2)
    lat = math.degrees(math.atan2(z, math.sqrt(x**2 + y**2)))
    lon = math.degrees(math.atan2(y, x))
    alt = r - R_earth
    return lat, lon, alt

def compute_angles(x, y, z):
    phi = math.degrees(math.atan2(y, x))
    theta = math.degrees(math.atan2(z, math.sqrt(x**2 + y**2)))
    return phi, theta

def follow(file):
    file.seek(0, os.SEEK_END)
    while True:
        line = file.readline()
        if not line:
            time.sleep(0.2)
            continue
        yield line

def unwrap_phase(phases):
    unwrapped = [phases[0]]
    for p in phases[1:]:
        diff = p - unwrapped[-1]
        if diff > 180: p -= 360
        elif diff < -180: p += 360
        unwrapped.append(p)
    return unwrapped

def main():
    orbit_re   = re.compile(r"Position = \(([^,]+), ([^,]+), ([^)]+)\).*Velocity = \(([^,]+), ([^,]+), ([^)]+)\)")
    doppler_re = re.compile(r"Doppler shift UE->SAT ([\d\.\-]+) kHz")
    delay_re   = re.compile(r"Uplink delay ([\d\.]+) ms")
    vel_acc_re = re.compile(r"velocity towards gNB: ([\d\.\-]+) m/s, acceleration towards gNB: ([\d\.\-]+)")

    with open(CSV_FILE, "w", newline="") as f:
        csv.writer(f).writerow(["Time(s)","x","y","z","vx","vy","vz",
                                "Lat","Lon","Alt","Phi","Theta","Phase",
                                "Delay","Doppler","Vel_toward_gNB","Acc_toward_gNB"])

    plt.ion()
    fig = plt.figure(figsize=(12,10))
    ax3d = fig.add_subplot(3,2,1, projection='3d')
    ax_phase = fig.add_subplot(3,2,2)
    ax_delay = fig.add_subplot(3,2,3)
    ax_dopp  = fig.add_subplot(3,2,4)
    ax_phi   = fig.add_subplot(3,2,5)
    ax_theta = fig.add_subplot(3,2,6)
    fig.suptitle("5G NTN Live Telemetry Dashboard", fontsize=14)

    R_e = 6371000
    u,v = np.mgrid[0:2*np.pi:40j, 0:np.pi:20j]
    xE = R_e*np.cos(u)*np.sin(v)
    yE = R_e*np.sin(u)*np.sin(v)
    zE = R_e*np.cos(v)
    ax3d.plot_wireframe(xE,yE,zE,color="lightgray",linewidth=0.3)
    ax3d.set_title("Satellite Orbit (ECI)")
    ax3d.set_xlabel("X (m)"); ax3d.set_ylabel("Y (m)"); ax3d.set_zlabel("Z (m)")
    ax3d.set_box_aspect([1,1,1])

    track3d, = ax3d.plot([],[],[],'r-')
    sat3d,   = ax3d.plot([],[],[],'bo')
    line_phase, = ax_phase.plot([],[],'m-')
    line_delay, = ax_delay.plot([],[],'g-')
    line_dopp,  = ax_dopp.plot([],[],'r-')
    line_phi,   = ax_phi.plot([],[],'c-')
    line_theta, = ax_theta.plot([],[],'y-')

    for ax in [ax_phase,ax_delay,ax_dopp,ax_phi,ax_theta]: ax.grid(True)

    xs,ys,zs,phases,delays,dopplers,phis,thetas=[],[],[],[],[],[],[],[]

    if not os.path.exists(LOG_FILE):
        print(f"❌ Log file not found: {LOG_FILE}")
        return

    with open(LOG_FILE,"r") as f:
        for line in follow(f):
            updated=False; tstamp=time.time()
            if "Satellite orbit" in line:
                m=orbit_re.search(line)
                if m:
                    x,y,z,vx,vy,vz=map(float,m.groups())
                    xs.append(x); ys.append(y); zs.append(z)
                    phi,theta=compute_angles(x,y,z)
                    phis.append(phi); thetas.append(theta)
                    phase=math.degrees(math.atan2(y,x))
                    phases.append(phase)
                    lat,lon,alt=eci_to_geo(x,y,z)
                    csv.writer(open(CSV_FILE,"a",newline="")).writerow(
                        [tstamp,x,y,z,vx,vy,vz,lat,lon,alt,phi,theta,phase,"","","",""])
                    updated=True
            if "Uplink delay" in line:
                m=delay_re.search(line)
                if m:
                    d=float(m.group(1))
                    delays.append(d); line_delay.set_data(range(len(delays)),delays)
                    csv.writer(open(CSV_FILE,"a",newline="")).writerow(["","","","","","","","","","","","","","",d,"","",""])
                    updated=True
            if "Doppler shift UE->SAT" in line:
                m=doppler_re.search(line)
                if m:
                    dp=float(m.group(1))
                    dopplers.append(dp); line_dopp.set_data(range(len(dopplers)),dopplers)
                    csv.writer(open(CSV_FILE,"a",newline="")).writerow(["","","","","","","","","","","","","","","",dp,"",""])
                    updated=True
            if "velocity towards gNB" in line:
                m=vel_acc_re.search(line)
                if m:
                    vg,ag=map(float,m.groups())
                    csv.writer(open(CSV_FILE,"a",newline="")).writerow(["","","","","","","","","","","","","","","","","",vg,ag])
                    updated=True
            if phases:
                pu=unwrap_phase(phases)
                line_phase.set_data(range(len(pu)),pu)
                line_phi.set_data(range(len(phis)),phis)
                line_theta.set_data(range(len(thetas)),thetas)
            if updated:
                for ax in [ax_phase,ax_delay,ax_dopp,ax_phi,ax_theta]:
                    ax.relim(); ax.autoscale_view()
                ax3d.relim(); ax3d.autoscale_view()
                plt.draw(); plt.pause(0.05)

    plt.ioff(); plt.show()

if __name__=="__main__": main()

🛰 Result

Run all three terminals; the dashboard will show the live orbit and telemetry while ntn_data.csv logs every value for offline analysis.
