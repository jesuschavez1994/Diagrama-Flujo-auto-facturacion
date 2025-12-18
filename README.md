# Diagrama de flujo del portal de autofacturacion

A continuación se presentan diagramas Mermaid compatibles con GitHub. No hay comillas ni parentesis en labels. Cada subflujo cubre escenarios clave.

## Configuración de Git

Para evitar el error "fatal: Need to specify how to reconcile divergent branches" al hacer pull, configura tu repositorio local:

```bash
# Opción 1: Usar el archivo de configuración incluido
# (El path es relativo al directorio .git)
git config --local include.path ../.gitconfig

# Opción 2: Configurar manualmente (recomendado)
git config pull.rebase false  # usa merge al hacer pull
```

O para todos tus repositorios:
```bash
git config --global pull.rebase false
```

## Flujo principal desde ticket hasta entrega

```mermaid
flowchart TD
  %% Secciones
  subgraph Cliente
    A1[Accede al portal]
    A2[Ingresa ticket numero fecha total sucursal o QR]
    A3[Captura datos fiscales RFC nombre regimen CP usoCFDI email]
    A4[Confirma y acepta privacidad y terminos]
  end

  subgraph Frontend
    F1[Render wizard y activar captcha invisible]
    F2[Enviar lookup con captchaToken]
    F3[Validar RFC CP y combinaciones basicas]
    F4[Solicitar previsualizacion]
    F5[Mostrar resumen de conceptos impuestos y total]
    F6[Generar Idempotency Key]
    F7[Llamar emision issue con ticketRef receptor y email]
    F8[Mostrar exito con UUID y links firmados de XML y PDF]
    F9[Mostrar error y guia de correccion]
    F10[Desafio captcha explicito si hay abuso]
  end

  subgraph Backend
    B1[Validar ticket en POS o ERP y normalizar datos]
    B2[Calcular previsualizacion totales impuestos metodo y forma]
    B3[Construir XML CFDI 4.0 con conceptos y receptores]
    B4[Firmar con CSD del emisor o usar custodia en PAC]
    B5[Clasificar errores y mapear codigos para UI]
  end

  subgraph POS_ERP
    P1[Buscar ticket y estado]
  end

  subgraph PAC
    C1[Timbrar CFDI enviar XML]
    C2[Responder UUID y timbre o error]
  end

  subgraph SAT
    S1[Consulta CFDI opcional para verificacion]
  end

  subgraph Storage
    ST1[Guardar XML y PDF en almacenamiento seguro]
    ST2[Generar links firmados con expiracion]
  end

  subgraph Email
    E1[Enviar correo con links o adjuntos]
  end

  %% Flujo inicial
  A1 --> F1
  F1 --> A2
  A2 --> F2
  F2 --> B1
  B1 --> P1
  P1 --> B1

  %% Decisiones de elegibilidad
  B1 -->|Existe| D1{Ticket devuelto o cancelado}
  B1 -->|No existe| X1[Error ticket no encontrado]
  X1 --> F9

  D1 -->|Si| X2[Bloqueado por devolucion o cancelacion]
  X2 --> F9
  D1 -->|No| D2{Ticket ya facturado}

  D2 -->|Si| X3[Ofrecer redescarga de CFDI vigente]
  X3 --> F9
  D2 -->|No| D3{Ticket en factura global}

  D3 -->|Si| X4[Bloqueado por factura global con opcion de solicitud de liberacion]
  X4 --> F9
  D3 -->|No| D4{Dentro de ventana permitida}

  D4 -->|No| X5[Fuera de ventana mostrar mensaje y canal de soporte]
  X5 --> F9
  D4 -->|Si| A3

  %% Captura y previsualizacion
  A3 --> F3
  F3 -->|Valido| F4
  F3 -->|Invalido| F9
  F4 --> B2
  B2 --> F5
  F5 --> A4

  %% Emision
  A4 --> F6
  F6 --> F7
  F7 --> B3
  B3 --> B4
  B4 --> C1
  C1 --> C2

  %% Respuesta del PAC
  C2 -->|Exito| ST1
  C2 -->|Error| B5
  B5 --> F9

  %% Entrega
  ST1 --> ST2
  ST2 --> F8
  ST2 --> E1
  E1 --> F8

  %% Verificacion opcional
  ST1 -.-> S1
```

Leyenda rapida
- Ya facturado significa que el ticket tiene un CFDI vigente con UUID asignado
- En factura global significa que el ticket fue incluido en un CFDI a publico en general y se bloquea la emision individual
- Ventana permitida es el periodo definido por negocio para poder emitir la factura individual
- Idempotency Key previene facturas duplicadas por doble clic

## Flujo de re descarga de CFDI

```mermaid
flowchart TD
  subgraph Cliente
    R1[Abre pagina de redescarga]
    R2[Ingresa ticket y RFC o token de ticket]
  end

  subgraph Frontend
    RF1[Validar formato de RFC y datos basicos]
    RF2[Enviar solicitud de redescarga]
    RF3[Mostrar links firmados o enviar por correo]
    RF4[Mostrar error claro si no coincide]
  end

  subgraph Backend
    RB1[Validar que el ticket tiene CFDI vigente]
    RB2[Verificar coincidencia de RFC y ticket]
  end

  subgraph Storage
    RS1[Generar nuevos links firmados con expiracion]
  end

  R1 --> R2
  R2 --> RF1
  RF1 -->|Valido| RF2
  RF1 -->|Invalido| RF4
  RF2 --> RB1
  RB1 -->|No vigente| RF4
  RB1 -->|Vigente| RB2
  RB2 -->|Coincide| RS1
  RB2 -->|No coincide| RF4
  RS1 --> RF3
```

Mensajes recomendados
- Si no coincide RFC con el CFDI vigente mostrar mensaje de seguridad sin revelar datos
- Si el link expira permitir regenerar con validacion de ticket y RFC

## Flujo de cancelacion y sustitucion desde backoffice

```mermaid
flowchart TD
  subgraph Backoffice
    K1[Seleccionar CFDI por UUID]
    K2[Elegir motivo de cancelacion]
    K3[Confirmar solicitud]
    K4[Emitir CFDI sustituto si aplica]
  end

  subgraph Backend
    KB1[Validar estatus vigente y reglas de cancelacion]
    KB2[Enviar solicitud de cancelacion al PAC o al SAT]
    KB3[Actualizar estatus y acuse]
    KB4[Relacionar CFDI 04 si hay sustitucion]
  end

  subgraph PAC
    KC1[Procesar cancelacion]
    KC2[Responder acuse aceptado o rechazado]
  end

  subgraph SAT
    KS1[Consulta CFDI para confirmar estatus]
  end

  K1 --> K2
  K2 --> K3
  K3 --> KB1
  KB1 -->|No cancelable| XK1[Requiere aceptacion del receptor o fuera de plazo]
  XK1 --> K1
  KB1 -->|Cancelable| KB2
  KB2 --> KC1
  KC1 --> KC2
  KC2 -->|Aceptado| KB3
  KC2 -->|Rechazado| KB3
  KB3 --> KS1
  KB3 --> K4
  K4 --> KB4
```

Notas
- Cancelacion aceptada pasa a estatus cancelado
- Si hubo error de datos del receptor se emite un CFDI sustituto y se relaciona con clave 04
- El portal publico no expone cancelacion esto se opera en backoffice
