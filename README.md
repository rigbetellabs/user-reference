# RigBetel Labs User Reference

Public user reference for **RigBetel Labs** research robots: **Acrux**, **Cepheus**, and **Diadem**.

This repository contains user guides, topic references, and FAQs to help you get started with RigBetel Labs robots and the RigBetel Labs Standard Firmware.

---

## Documentation

| Document | Description |
|----------|-------------|
| [User Guide](docs/user_guide.md) | Complete operational manual — boot sequence, ROS 2 integration, motor control, battery management, safety, and troubleshooting |
| [ROS Topic Reference](docs/user_docs.md) | Detailed reference for all ROS 2 subscriber and publisher topics exposed by the firmware |
| [FAQ](docs/FAQ.md) | Frequently asked questions covering display, battery, navigation, PID, RC control, and more |

---

## Educational Platforms

The `educational/` folder is a placeholder for upcoming firmware and resources for the Tortoisebot educational robot series.

| Platform | Description |
|----------|-------------|
| [tortoisebot](https://github.com/rigbetellabs/tortoisebot) | Open Source educational robot (https://github.com/rigbetellabs/tortoisebot) |
| [ttb-pro](educational/ttb-pro/) | TTB-Pro educational robot  *(Docs coming soon)* |
| [ttb-pro-wifi](educational/ttb-pro-wifi/) | TTB-Pro WiFi variant with LIDAR support *(Docs coming soon)* |
| [ttb-pro-max](educational/ttb-pro-max/) | TTB-Pro Max educational robot  *(Docs coming soon)* |

---

## Supported Robots

| Robot | Drive Type | ROS Namespace |
|-------|-----------|---------------|
| Acrux | 2-wheel differential | `/acrux` `/cepheus`|| 
| Diadem | 4-wheel | `/diadem` `/cepheus`|

All robots run the **RigBetel Labs Standard Firmware** on an ESP32 microcontroller via the micro-ROS framework.

---

© RigBetel Labs
