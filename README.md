# AWS Hub & Spoke — Network Discovery & Best Practices Audit

[![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)
[![Boto3](https://img.shields.io/badge/boto3-latest-orange.svg)](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Script de descubrimiento profundo y auditoría automatizada de arquitecturas de red **Hub & Spoke** en AWS. Genera inventarios CSV, reportes de mejores prácticas y diagramas Mermaid.js de la topología descubierta.

> 🔗 Proyecto hermano: [azure-net-discovery](https://github.com/garymayet/azure-net-discovery) — Equivalente para entornos Azure.

---

## Qué hace este script

El script ejecuta 5 pilares de análisis sobre tu infraestructura de red AWS:

| Pilar | Descripción |
|-------|-------------|
| **1. Descubrimiento** | Itera cuentas y regiones recolectando VPCs, Subnets, Route Tables, Transit Gateways, TGW Attachments, VPC Peerings, VGWs, CGWs, VPN Connections, Direct Connect, Network Firewalls, Route 53 Resolver Endpoints y NAT Gateways. |
| **2. Clasificación** | Clasifica automáticamente cada VPC como **Hub** o **Spoke** mediante heurísticas basadas en la presencia de Network Firewall, Appliance Mode, NAT Gateways centralizados, R53 Resolver y VGWs. |
| **3. Best Practices** | Ejecuta 6 validaciones automatizadas con resultados PASS / FAIL / WARNING exportados a CSV. |
| **4. Diagrama Mermaid** | Genera un archivo `.mmd` con la topología completa lista para renderizar en [mermaid.live](https://mermaid.live) o GitHub. |
| **5. Exportación** | Produce CSVs de inventario por recurso, reporte de auditoría y un resumen arquitectónico en Markdown. |

### Validaciones de mejores prácticas incluidas

| Check | Qué valida |
|-------|-----------|
| `Spoke-Connectivity` | Spokes enrutados al TGW/Hub con ruta default `0.0.0.0/0` |
| `Spoke-to-Spoke-Peering` | Peerings directos entre Spokes que evaden el firewall central |
| `TGW-Attachment-HA` | Attachments del Transit Gateway en ≥2 Availability Zones |
| `NFW-Hub-HA` / `NAT-Hub-HA` | Network Firewall y NAT Gateways del Hub en múltiples AZs |
| `TGW-ApplianceMode` | Appliance Mode habilitado en la VPC de inspección (simetría de tráfico) |
| `VPN-TunnelRedundancy` / `DX-Redundancy` | Redundancia en túneles VPN y conexiones Direct Connect |

---

## Requisitos

- **Python 3.9+**
- **boto3** (preinstalado en AWS CloudShell)
- Credenciales AWS con permisos de lectura sobre los recursos de red (ver [Permisos IAM mínimos](#permisos-iam-mínimos))

---

## Ejecución en AWS CloudShell (Método recomendado)

AWS CloudShell es la forma más directa de ejecutar el script, especialmente en entornos con autenticación federada (OKTA/SAML/SSO). La sesión hereda automáticamente las credenciales del usuario autenticado en la consola — no se requiere `aws configure` ni extracción manual de tokens.

### Paso 1 — Abrir CloudShell

1. Iniciar sesión en la **AWS Management Console** de la cuenta objetivo.
2. Hacer clic en el ícono de terminal (**`>_`**) en la barra superior, o navegar a: `https://console.aws.amazon.com/cloudshell/`
3. Esperar a que el entorno se inicialice (primera vez toma ~30 segundos).

### Paso 2 — Descargar el script

**Opción A** — Clonar desde GitHub:

```bash
git clone https://github.com/garymayet/aws-net-discovery.git ~/aws-net-discovery
```

**Opción B** — Descarga directa (sin git):

```bash
curl -sL https://raw.githubusercontent.com/garymayet/aws-net-discovery/main/discover_hub_spoke_aws.py \
  -o ~/discover_hub_spoke_aws.py
```

**Opción C** — Subir manualmente:

1. En CloudShell, hacer clic en **Actions → Upload file**.
2. Seleccionar el archivo `discover_hub_spoke_aws.py` desde tu máquina local.
3. El archivo se sube a `~/discover_hub_spoke_aws.py`.

> **Nota:** El directorio `$HOME` en CloudShell es persistente (hasta 1 GB). El script permanece entre sesiones — no es necesario volver a subirlo cada vez.

### Paso 3 — Ejecutar el script

```bash
# Cuenta única — regiones específicas
python3 ~/discover_hub_spoke_aws.py \
  --regions us-east-1,us-east-2,sa-east-1 \
  --output ~/hub-spoke-output
```

El script mostrará progreso en consola con códigos de color:

```
[14:32:01] [INFO ] ═══════════════════════════════════════════════════════════
[14:32:01] [INFO ]   AWS Hub & Spoke — Deep Discovery & Best Practices Audit
[14:32:01] [INFO ] ═══════════════════════════════════════════════════════════
[14:32:02] [OK   ]   Usando credenciales por defecto → Cuenta 123456789012
[14:32:02] [INFO ]   Región: us-east-1
[14:32:02] [INFO ]     Descubriendo VPCs...
...
[14:33:15] [OK   ]   Hubs: 1 | Spokes: 4 | Sin clasificar: 0
...
[14:33:45] [OK   ]   RESUMEN: 12 PASS | 2 FAIL | 3 WARNINGS
```

### Paso 4 — Descargar los resultados

**Opción A** — Descargar archivo individual:

1. Hacer clic en **Actions → Download file**.
2. Escribir la ruta del archivo:
   - `~/hub-spoke-output/best-practices-report.csv`
   - `~/hub-spoke-output/hub-spoke-topology.mmd`
   - `~/hub-spoke-output/architecture-summary.md`

**Opción B** — Descargar todo comprimido:

```bash
cd ~ && tar czf hub-spoke-results.tar.gz hub-spoke-output/
```

Luego: **Actions → Download file** → `~/hub-spoke-results.tar.gz`

---

## Ejecución multi-cuenta

### Con rol cross-account (recomendado)

Si tienes un rol IAM desplegado en todas las cuentas objetivo que permita `sts:AssumeRole` desde tu cuenta actual:

```bash
python3 ~/discover_hub_spoke_aws.py \
  --account-ids 111111111111,222222222222,333333333333 \
  --role-arn "arn:aws:iam::{account_id}:role/NetworkAudit-ReadOnly" \
  --regions us-east-1,sa-east-1 \
  --output ~/hub-spoke-output
```

El placeholder `{account_id}` se reemplaza automáticamente por cada ID de cuenta.

### Con AWS Organizations

Si tu cuenta tiene acceso a `organizations:ListAccounts`:

```bash
python3 ~/discover_hub_spoke_aws.py \
  --org \
  --role-arn "arn:aws:iam::{account_id}:role/OrganizationAccountAccessRole" \
  --regions us-east-1 \
  --output ~/hub-spoke-output
```

### Con perfiles de AWS CLI (fuera de CloudShell)

Para ejecución local con `~/.aws/credentials`:

```bash
python3 discover_hub_spoke_aws.py \
  --profiles produccion,desarrollo,staging \
  --regions us-east-1,us-west-2 \
  --output ./hub-spoke-output
```

### Sin rol cross-account (cuenta por cuenta)

Si no hay un rol centralizado disponible, se puede ejecutar el script desde CloudShell en cada cuenta por separado:

```bash
#!/bin/bash
# Ejecutar en CloudShell de cada cuenta
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

python3 ~/discover_hub_spoke_aws.py \
  --regions us-east-1,sa-east-1 \
  --output ~/hub-spoke-output-${ACCOUNT_ID}

tar czf ~/results-${ACCOUNT_ID}.tar.gz ~/hub-spoke-output-${ACCOUNT_ID}/
echo "Listo: ~/results-${ACCOUNT_ID}.tar.gz"
```

Para consolidar los CSVs después de descargar todos los resultados:

```bash
# En tu máquina local
head -1 hub-spoke-output-111111111111/inventory-vpcs.csv > consolidated-vpcs.csv
for d in hub-spoke-output-*/; do
  tail -n +2 "${d}inventory-vpcs.csv" >> consolidated-vpcs.csv
done
```

---

## Archivos de salida

Después de la ejecución, el directorio de salida contiene:

```
hub-spoke-output/
├── best-practices-report.csv        # Auditoría PASS/FAIL/WARNING
├── hub-spoke-topology.mmd           # Diagrama Mermaid.js
├── architecture-summary.md          # Resumen completo en Markdown
├── inventory-vpcs.csv               # Inventario de VPCs (Hub/Spoke)
├── inventory-subnets.csv            # Inventario de Subnets
├── inventory-route-tables.csv       # Route Tables con rutas
├── inventory-tgws.csv               # Transit Gateways
├── inventory-tgw-attachments.csv    # TGW Attachments (AZs, Appliance Mode)
├── inventory-vpc-peerings.csv       # VPC Peerings
├── inventory-vgws.csv               # Virtual Private Gateways
├── inventory-cgws.csv               # Customer Gateways
├── inventory-vpn-connections.csv    # VPN Connections + estado de túneles
├── inventory-dx-connections.csv     # Direct Connect Connections
├── inventory-network-firewalls.csv  # AWS Network Firewalls
├── inventory-r53-resolver.csv       # Route 53 Resolver Endpoints
└── inventory-nat-gateways.csv       # NAT Gateways
```

Para visualizar el diagrama Mermaid: copiar el contenido de `hub-spoke-topology.mmd` y pegarlo en [mermaid.live](https://mermaid.live), o visualizar directamente en GitHub (GitHub renderiza bloques `mermaid` de forma nativa en archivos Markdown).

---

## Permisos IAM mínimos

El script opera en modo **solo lectura**. El siguiente JSON de política IAM contiene los permisos mínimos requeridos:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "NetworkDiscoveryReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeRouteTables",
        "ec2:DescribeTransitGateways",
        "ec2:DescribeTransitGatewayAttachments",
        "ec2:DescribeTransitGatewayVpcAttachments",
        "ec2:DescribeVpcPeeringConnections",
        "ec2:DescribeVpnGateways",
        "ec2:DescribeCustomerGateways",
        "ec2:DescribeVpnConnections",
        "ec2:DescribeNatGateways",
        "directconnect:DescribeConnections",
        "network-firewall:ListFirewalls",
        "network-firewall:DescribeFirewall",
        "route53resolver:ListResolverEndpoints",
        "route53resolver:ListResolverEndpointIpAddresses",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

Para ejecución **multi-cuenta**, agregar:

```json
{
  "Sid": "CrossAccountAssume",
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::*:role/NetworkAudit-ReadOnly"
}
```

Para ejecución con **AWS Organizations**, agregar:

```json
{
  "Sid": "OrganizationsRead",
  "Effect": "Allow",
  "Action": "organizations:ListAccounts",
  "Resource": "*"
}
```

> **Nota:** Si algún permiso falta, el script no se detiene. Captura el `AccessDeniedException`, emite un `WARNING` en consola, y continúa con el siguiente recurso.

---

## Consideraciones de CloudShell

| Aspecto | Detalle |
|---------|---------|
| **Timeout** | La sesión se cierra tras 20 minutos de inactividad. El script produce output constante en consola, lo que mantiene la sesión activa. |
| **Almacenamiento** | Solo `$HOME` es persistente (máx. 1 GB). Los resultados se almacenan ahí. |
| **Red** | Las llamadas a la API de AWS usan los endpoints internos — no hay restricciones de egress para las APIs utilizadas por el script. |
| **Python/boto3** | Preinstalados. No se requiere `pip install` adicional. |
| **Regiones** | CloudShell está disponible en la mayoría de las regiones comerciales. El script puede consultar cualquier región independientemente de en cuál se abrió CloudShell. |

---

## Referencia rápida de uso

```bash
# Ejecución básica — cuenta actual, regiones por defecto
python3 discover_hub_spoke_aws.py

# Regiones específicas
python3 discover_hub_spoke_aws.py --regions us-east-1,sa-east-1

# Multi-cuenta con rol cross-account
python3 discover_hub_spoke_aws.py \
  --account-ids 111111111111,222222222222 \
  --role-arn "arn:aws:iam::{account_id}:role/AuditRole"

# AWS Organizations
python3 discover_hub_spoke_aws.py --org \
  --role-arn "arn:aws:iam::{account_id}:role/OrgAccessRole"

# Múltiples perfiles locales
python3 discover_hub_spoke_aws.py --profiles prod,dev,staging

# Output personalizado
python3 discover_hub_spoke_aws.py --output ~/mis-resultados
```

---

## Licencia

MIT

---

*Desarrollado como herramienta de gobernanza de red para entornos AWS multi-cuenta con arquitectura Hub & Spoke.*
