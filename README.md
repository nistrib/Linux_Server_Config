# Ubuntu 24.04 LTS Secure Server Setup

A complete, production-ready setup guide for Ubuntu 24.04 LTS with security-first architecture, Python data science environment, and web-based development tools.

## 🚀 Quick Overview

This guide sets up a secure Ubuntu server with:

- **🔒 Security**: Tailscale VPN + nftables firewall + fail2ban
- **💻 Development**: VS Code Server + JupyterLab  
- **📊 Data Science**: Python 3 + NumPy, Pandas, scikit-learn, PyTorch
- **🖥️ Management**: Cockpit web interface
- **📁 File Sharing**: Samba (Windows/Mac/Linux)
- **💾 Backups**: Automated hourly backups (15-day retention)

**All services accessible only via Tailscale VPN - Zero public exposure!**

## 📋 What You Get

| Service | Port | Access |
|---------|------|--------|
| VS Code Server | 8080 | Browser-based IDE |
| JupyterLab | 8888 | Interactive notebooks |
| Cockpit | 9090 | Server management UI |
| Samba | 445 | File sharing |

## ⚡ Quick Start

```bash
# 1. Fresh Ubuntu 24.04 LTS installation
# 2. Follow **Linux Server Full Config.txt** step-by-step
# 3. Access your services via Tailscale VPN
```

**Installation time:** ~30 minutes

## 📖 Documentation

- Linux_Server_Configuration.txt - Full server setup documentation (Complete step-by-step guide)
- All commands are copy-paste ready
- Includes troubleshooting section
- Security hardening recommendations

## 🔐 Security Features

✅ Tailscale zero-trust VPN (WireGuard encryption)  
✅ nftables firewall (blocks all public internet traffic)  
✅ fail2ban intrusion prevention  
✅ No public SSH access  
✅ Automated backups  
✅ Automatic security updates  

**Important:** Enable 2FA on your Tailscale account!

## 📦 Prerequisites

- Ubuntu 24.04 LTS (fresh install recommended)
- Root or sudo access
- Tailscale account (free at [tailscale.com](https://tailscale.com))
- Basic Linux command line knowledge

## 🛠️ Installed Software

**Security:** Tailscale, nftables, fail2ban  
**Development:** VS Code Server, JupyterLab, Python 3  
**Management:** Cockpit, Cockpit Navigator, Cockpit Files  
**File Sharing:** Samba  
**Backups:** Timeshift with automated scripts  
**Data Science:** NumPy, Pandas, Matplotlib, Seaborn, scikit-learn, PyTorch  

## 🎯 Use Cases

- Personal data science workstation
- Remote development environment
- Home lab server
- Secure research environment
- Private cloud storage
- Machine learning projects

## ⚠️ Important Notes

- **Change all default passwords** before use
- **Enable 2FA** on your Tailscale account (critical!)
- Test in non-production environment first
- Review security recommendations in INSTALLATION.txt

## 🤝 Contributing

Found an issue or have improvements? 

- Open an issue on GitHub
- Submit a pull request
- Share your feedback

## 📄 License

MIT License - Free to use, modify, and distribute.

See LICENSE file for full license text.

## 🙏 Acknowledgments

Built with open-source tools:
- [Tailscale](https://tailscale.com) - Zero-trust VPN
- [code-server](https://github.com/coder/code-server) - VS Code in browser
- [JupyterLab](https://jupyter.org) - Interactive computing
- [Cockpit](https://cockpit-project.org) - Server management

## 📞 Support

- Check Linux Server Full Config.txt troubleshooting section
- Review security recommendations
- Open GitHub issue for bugs/questions

## ⚡ Quick Commands

```bash
# Check all services status
systemctl is-active nftables tailscaled cockpit.socket smbd jupyter code-server@$USER fail2ban

# View your Tailscale IP
tailscale ip -4

# Monitor firewall
sudo journalctl -kf | grep "nft-drop"

# Check backups
sudo timeshift --list

# View fail2ban status
sudo fail2ban-client status
```

---

**Made with ❤️ for secure, private development environments**
