# Templates Zabbix para provedores e redes sem fio

Templates para **Zabbix 7.4** feitos para monitorar equipamentos comuns em provedores e redes municipais: rádios **Ubiquiti** (airMAX M5 e AC, airFiber 5XHD), **OLT GPON TP-Link DS-P7001-16** e switches **Intelbras S3000**.

Eles nasceram da operação de uma rede sem fio real, organizada em POPs. Os templates genéricos que existiam não traziam o que importa no dia a dia (sinal, ruído, SNR, CCQ, estações conectadas, capacidade do enlace, potência óptica das ONUs, fontes e ventilador), então criei estes. Todos estão em uso em produção.

| Pasta | Template | Para quê |
|---|---|---|
| `ubiquiti/ubiquiti-airmax.yaml` | Template Wireless Ubiquiti Base | Ping, perda, latência, porta eth0 e uptime |
| | Template Ubiquiti AP | Base + número de estações conectadas |
| | Template Ubiquiti RF M5 | Sinal, ruído, SNR, CCQ e taxas (airMAX M) |
| | Template Ubiquiti RF AC | Sinal, ruído, SNR e taxas (airMAX AC) |
| `ubiquiti/ubiquiti-airfiber-5xhd.yaml` | Template Ubiquiti airFiber 5XHD | Descoberta dos enlaces: sinal local/remoto, capacidade, modulação, distância |
| `olt/tplink-ds-p7001-16.yaml` | TP-Link DS-P7001-16 by SNMP | Descoberta de PONs e ONUs: online/offline, potência RX/TX, temperatura, tensão, distância, serial |
| `intelbras/intelbras-s3000-base.yaml` | Template Intelbras S3000 by SNMP | Interfaces físicas, sistema e uptime (herda o oficial *Network Generic Device by SNMP*) |
| `intelbras/intelbras-s3000-fw-9.7.200.yaml` | Template Intelbras S3000 MIB 9.7.200 | Memória (único dado que esse firmware expõe) |
| `intelbras/intelbras-s3000-fw-9.7.400.yaml` | Template Intelbras S3000 MIB 9.7.400 | CPU, memória, temperaturas, fontes e ventilador |

## Como importar

1. No Zabbix, vá em **Coleta de dados → Modelos → Importar**.
2. Escolha o arquivo `.yaml` e confirme.
3. No host, use interface **SNMP** e coloque as credenciais **no host**, nunca no template: a community na macro `{$SNMP_COMMUNITY}` (v1/v2c) ou usuário e senhas do SNMPv3 na interface. A OLT foi validada com SNMPv3.

Combinações usadas na prática:

| Equipamento | Templates no host |
|---|---|
| AP airMAX M (Rocket M5, NanoStation M5…) | Template Ubiquiti AP + Template Ubiquiti RF M5 |
| AP airMAX AC (Rocket 5AC, LiteBeam AC…) | Template Ubiquiti AP + Template Ubiquiti RF AC |
| Estação airMAX | Template Wireless Ubiquiti Base + RF M5 ou RF AC |
| airFiber 5XHD / AF-LTU | Template Ubiquiti airFiber 5XHD |
| OLT TP-Link / DeltaStream DS-P7001-16 | TP-Link DS-P7001-16 by SNMP |
| Switch Intelbras S3054G-B (fw 9.7.400) | Template Intelbras S3000 by SNMP + Template Intelbras S3000 MIB 9.7.400 |
| Switch Intelbras S3028G-B (fw 9.7.200) | Template Intelbras S3000 by SNMP + Template Intelbras S3000 MIB 9.7.200 |

No airOS, ative o SNMP em **Services → SNMP Agent**.

## Alertas e limites

Todos os limites são **macros**. Ajuste no template para a rede inteira ou no host para um equipamento específico.

| Macro | Padrão | Significado |
|---|---|---|
| `{$ICMP.LOSS.MAX}` | 20 | Perda de pacotes máxima (%) |
| `{$ICMP.RESPONSE.MAX}` | 0.2 | Latência máxima (s) |
| `{$RSSI.WARN}` | −70 (M5) / −67 (AC) | Sinal de atenção (dBm) |
| `{$RSSI.MIN}` | −75 | Sinal crítico (dBm) |
| `{$NOISE.MAX}` | −85 | Ruído máximo (dBm) |
| `{$SNR.MIN}` | 20 | SNR mínimo (dB) |
| `{$CCQ.MIN}` | 75 | CCQ mínimo (%, só M5) |
| `{$AF.SINAL.MIN}` | −70 | Sinal mínimo do enlace airFiber (dBm) |
| `{$AF.CAP.MIN}` | 50M | Capacidade mínima do enlace airFiber |
| `{$ONU.RX.WARN}` / `{$ONU.RX.CRIT}` | −27 / −29 | Potência RX da ONU: atenção / crítico (dBm) |
| `{$ONU.TEMP.WARN}` / `{$ONU.TEMP.CRIT}` | 75 / 85 | Temperatura da ONU (°C) |
| `{$ONU.OFFLINE.ALERT}` | 0 | Mude para 1 para alertar ONU offline (deixe 0 até limpar ONUs antigas da OLT) |
| `{$INTELBRAS.CPU.MAX}` / `MEM.MAX` | 80 / 90 | CPU e memória do switch (%) |
| `{$INTELBRAS.TEMP.AMB.MAX}` / `TEMP.CPU.MAX` | 50 / 85 | Temperaturas (°C) |

Decisões para evitar avalanche de alertas:
- Perda, latência e "LAN desconectada" **dependem** de "Host down": se o rádio cai, chega um alerta só.
- "RSSI baixo" depende de "RSSI crítico": nunca chegam os dois para o mesmo problema.
- "ONU offline" só dispara se a OLT continua respondendo. Se a OLT inteira para, chega só o alerta da OLT, e não um por ONU.
- "Qualidade baixa (SNR)" só dispara quando o sinal está bom e o SNR está ruim, o que normalmente indica **interferência no canal**.

**Dica:** se o próprio servidor Zabbix perder a rede, todos os hosts caem ao mesmo tempo. Crie um host "sentinela" (o gateway do Zabbix, por exemplo) e faça o trigger "Host down" depender dele.

## OIDs principais

| Dado | OID |
|---|---|
| Sinal / ruído / CCQ (airMAX) | `1.3.6.1.4.1.41112.1.4.5.1.5` / `.8` / `.7` |
| Taxa TX / RX (airMAX) | `1.3.6.1.4.1.41112.1.4.5.1.9` / `.10` |
| Estações conectadas | `1.3.6.1.4.1.41112.1.4.5.1.15` |
| Enlaces airFiber (descoberta) | `1.3.6.1.4.1.41112.1.10.1.4.1` |
| ONUs da OLT TP-Link (tabela) | `1.3.6.1.4.1.11863.6.100.1.7.2.1` (RX `.25`, TX `.26`, temperatura `.29`) |
| PONs da OLT TP-Link | `1.3.6.1.4.1.11863.6.100.1.1.2.1` |
| Intelbras S3000: CPU / memória / fontes | `1.3.6.1.4.1.26138.3.600.9` / `.10` / `.7` |

## Próximos

- [ ] MikroTik (complementos para os templates oficiais)
- [ ] Dashboards prontos

Testado no Zabbix 7.4. Encontrou um problema ou tem outro modelo? Abra uma *issue*.

## Licença

MIT. Use, modifique e compartilhe.

---
Feito por [Flávio Araújo](https://github.com/dectorblues) · [LinkedIn](https://linkedin.com/in/flavio51)
