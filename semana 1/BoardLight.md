***
# 1. Enumeración

Realizamos un escaneo a la máquina vulnerable para ver posibles puertos abiertos

```bash
sudo nmap -p- --open -sV -sS --min-rate 5000 -n 10.10.11.11 -oN scanPorts
```
