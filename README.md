# iac-universidad-hibrida

Infraestructura como código de la red híbrida de la Universidad Híbrida, escrita en **Ansible con YAML**.
Un mismo repositorio configura las dos mitades del caso: la nube en AWS y los equipos Cisco del campus.

## La idea central: los datos van separados del código

Las tres tablas del entregable viven en `group_vars/all/` como archivos YAML. Los roles no llevan
direcciones ni reglas escritas: solo leen esos archivos.

| Archivo | Qué contiene | Entregable del que sale |
|---|---|---|
| `addressing.yml` | VPC, subredes, VLAN, loopbacks y enlaces /31 | Tabla 2 · Subredes |
| `routing.yml` | ASN, sesiones BGP, tablas del Transit Gateway y de las VPC | Tabla 3 · Rutas |
| `flows.yml` | Los 24 flujos, con dónde se aplica cada uno | Matriz de flujos y controles |
| `common.yml` | Etiquetas obligatorias, dominios y destino de los registros | Observabilidad y costos |

Cambiar una subred o un flujo es editar una línea de YAML: el pipeline genera la configuración de AWS
y de los routers a partir de ahí. Así las tablas del documento y la red real no se desincronizan.

## Estructura

```
iac-universidad-hibrida/
├── site.yml                  # despliegue completo, en orden
├── audit.yml                 # compara lo desplegado con los YAML (detecta deriva)
├── inventory/hosts.yml       # AWS (API) y equipos del campus (SSH)
├── group_vars/
│   ├── all/                  # los datos: direccionamiento, rutas, flujos, etiquetas
│   └── campus/vault.yml      # credenciales, cifradas con ansible-vault
├── roles/
│   ├── aws_network/          # Transit Gateway, VPC, subredes, IGW, NAT y attachments
│   ├── aws_hybrid/           # Direct Connect gateway y VPN de respaldo
│   ├── aws_inspection/       # Network Firewall con las reglas de flows.yml
│   ├── aws_vpc_routes/       # tablas de rutas por tier y zona, endpoints de S3
│   ├── aws_tgw_routing/      # las siete tablas del Transit Gateway
│   ├── aws_security/         # security groups de flows.yml y VPC Flow Logs
│   ├── aws_dns/              # DNS híbrido (Resolver y zona privada)
│   ├── campus_network/       # interfaces, VLAN, SVI y OSPF (IOS y NX-OS)
│   └── campus_bgp/           # eBGP hacia AWS, prefix-lists y LOCAL_PREF
└── .github/workflows/        # lint, --check --diff, aplicación aprobada y auditoría diaria
```

## Cómo se ejecuta

```bash
ansible-galaxy collection install -r requirements.yml
pip install boto3 netaddr

ansible-playbook site.yml --check --diff    # qué cambiaría, sin aplicar
ansible-playbook site.yml --tags aws        # solo la nube
ansible-playbook site.yml --tags campus     # solo el campus
ansible-playbook audit.yml                  # verificar que la red coincide con los YAML
```

## Qué se hace con módulos y qué no

Ansible tiene módulos para casi todo el diseño. Para lo que no tiene módulo se usan plantillas
**CloudFormation en YAML**, que se generan desde los mismos datos y se despliegan con el módulo
`amazon.aws.cloudformation`, de forma declarativa e idempotente.

| Componente | Cómo se despliega |
|---|---|
| VPC, subredes, IGW, NAT, attachments, tablas de rutas de VPC, security groups | Módulos `amazon.aws` |
| Transit Gateway, customer gateway, VPN, Direct Connect gateway | Módulos `amazon.aws` y `community.aws` |
| Network Firewall, tablas del Transit Gateway, Route 53 Resolver, VPC Flow Logs | Plantillas CloudFormation (YAML) |
| Asociación del Direct Connect gateway al Transit Gateway | CLI de AWS, con una consulta previa para que sea idempotente |
| Routers y switches del campus | Módulos `cisco.ios` y `cisco.nxos` |

## Ansible frente a Terraform: qué se gana y qué se compensa

Se eligió Ansible porque es la herramienta que propone el curso y porque configura en un solo
repositorio tanto AWS como los equipos Cisco del campus, a partir de los mismos datos.

A cambio, Ansible no guarda estado como Terraform, y eso se compensa así:

1. **No hay `plan` real.** `--check --diff` muestra qué cambiaría sobre lo que ya existe, pero en el
   primer despliegue no puede anticipar los recursos que dependen de otros aún no creados.
2. **No detecta cambios hechos a mano.** Por eso existe `audit.yml`, que corre a diario y falla si
   una VPC o una subred ya no coincide con `addressing.yml`, o si el campus dejó de preferir el
   Direct Connect.
3. **El orden es explícito.** Cada rol usa los IDs que dejó el anterior, así que `site.yml` define
   la secuencia en lugar de un grafo de dependencias automático.

## Control de cambios con un equipo pequeño

Todo cambio entra por pull request. El pipeline valida el YAML, ejecuta `ansible-lint` y publica qué
cambiaría. Aplicar en AWS exige aprobación manual en el entorno `produccion` y termina con
`audit.yml`. No hay comité de cambios: el control es que nadie aplica sin que otra persona haya
revisado qué va a cambiar.

## Supuestos y límites de esta versión

- Todo se despliega en una sola cuenta para el laboratorio. En producción cada VPC va en su propia
  cuenta y el Transit Gateway se comparte con AWS RAM; los datos no cambian.
- IPv6 usa el prefijo de documentación 2001:db8::/32 (RFC 3849), que AWS no permite desplegar. En el
  laboratorio las VPC reciben un bloque IPv6 de Amazon y el código configura IPv4.
- La configuración IPsec de los túneles la genera AWS al crear la VPN y se aplica en R-EDGE-02 con el
  archivo que entrega AWS; este repositorio configura su direccionamiento y el BGP.
- El firewall del campus (Firepower) se gestiona desde su consola FMC y no está en el inventario.
- CloudFront, WAF, Identity Center y PrivateLink se despliegan con la aplicación y la gestión de
  identidad; en `flows.yml` aparecen como flujos documentados sin regla de red.

## Cómo se verificó

`yamllint` y `ansible-lint` pasan con el perfil `production`, y `ansible-playbook --syntax-check` pasa en
los dos playbooks. Las plantillas CloudFormation y las configuraciones de cada equipo se generaron
con IDs de prueba para comprobar su contenido. El repositorio no se ha ejecutado contra una cuenta
de AWS ni contra equipos Cisco reales.

## Documentacion

La documentacion se realizo con ayuda de Claude code para mantener el despliege lo mas facil posible