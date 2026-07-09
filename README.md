cat > README.md <<'EOF'
# Engineering Notes

Engineering notes for edge robotics infrastructure, Linux-based single-board computers, ROS networking, Wi-Fi mesh, and SwarmDock / SwarmPlug experiments.

This repository collects reproducible setup notes, troubleshooting records, and deployment procedures for lightweight edge nodes used in heterogeneous robotics and swarm systems.

## Current Notes

### Orange Pi Zero 3W

| Note | Topic |
|---|---|
| [Enable TUN kernel module for Tailscale](01-Hardware/OrangePi-Zero3W/01-orange-pi-zero-3w-enable-tun-for-tailscale.md) | Build and install `tun.ko` for the vendor `6.6.98-sun60iw2` kernel |
| [AR9271 USB Wi-Fi with IBSS + BATMAN-ADV mesh](01-Hardware/OrangePi-Zero3W/02-orange-pi-zero-3w-ar9271-ibss-batman-adv-mesh.md) | Use an AR9271 USB Wi-Fi adapter as an IBSS/ad-hoc underlay for BATMAN-ADV |

## Repository Scope

This repository focuses on:

- Linux single-board computer setup
- Kernel module adaptation
- Remote access and overlay networking
- Wi-Fi mesh and BATMAN-ADV experiments
- ROS / SwarmDock / SwarmPlug edge-node infrastructure

## Notes

These documents are engineering records based on specific hardware, kernel versions, and OS images. Commands may need adjustment for other boards, kernels, or distributions.

Do not blindly copy kernel modules across different kernel versions. Always verify:

```bash
uname -r
modinfo <module>.ko | grep vermagic
