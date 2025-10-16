# Guía de Integración de Wallets Crossmint

Guía de inicio rápido para crear wallets y enviar transacciones con el SDK de Crossmint.

## Tabla de Contenidos

- [Prerrequisitos](#prerrequisitos)
- [Instalación](#instalación)
- [Inicio Rápido](#inicio-rápido)
- [Ejemplos de Uso](#ejemplos-de-uso)
  - [Crear un Wallet](#crear-un-wallet)
  - [Enviar una Transacción](#enviar-una-transacción)
  - [Flujo Completo](#flujo-completo)
- [Configuración](#configuración)
- [Solución de Problemas](#solución-de-problemas)
- [Recursos](#recursos)

## Prerrequisitos

Antes de comenzar, asegúrate de tener:

- Node.js 16+ instalado
- Una cuenta de Crossmint con acceso a API
- Clave API con los siguientes scopes:
  - `wallets.create`
  - `wallets:transactions.create`

**Nota:** Para cadenas compatibles con EVM (ej. polygon), los wallets comparten un espacio de direcciones derivado de la misma clave privada, pero deben instanciarse por cadena para que las interacciones del SDK se enruten correctamente. Los Smart Wallets están soportados en muchas cadenas (ver abajo).

## Instalación
```bash
npm install @crossmint/server-sdk
```
Para codificación ERC-20 en ejemplos (opcional pero recomendado para transferencias de tokens):
```bash
npm install ethers
```

## Inicio Rápido
```typescript
import { createCrossmint, CrossmintWallets, EVMWallet } from "@crossmint/server-sdk";

const crossmint = createCrossmint({
    apiKey: "your-server-api-key-here",
    // environment: "staging" // Descomenta para staging (limita a testnets)
});

const crossmintWallets = CrossmintWallets.from(crossmint);
```

## Ejemplos de Uso

### Crear un Wallet
```typescript
async function createUserWallet(userEmail: string, chain: string = "polygon") {
    try {
        const wallet = await crossmintWallets.createWallet({
            chain,  // Ver cadenas soportadas; cadenas EVM comparten claves pero necesitan init por cadena
            signer: {
                type: "email",
                email: userEmail
            },
            // owner: `email:${userEmail}` // Opcional: Vincular a identidad de usuario para Crossmint Auth
        });

        console.log("✅ Wallet creado exitosamente!");
        console.log("Dirección:", wallet.address);
        return wallet;
    } catch (error) {
        console.error("❌ Error creando wallet:", error);
        throw error;
    }
}
```

**Ejemplo de Uso:**
```typescript
const wallet = await createUserWallet("user@globosend.com", "polygon");
```

### Enviar una Transacción
```typescript
import { ethers } from "ethers"; // Para codificar datos ERC-20

async function sendStablecoinTransfer(
    senderEmail: string,
    recipientAddress: string,
    amount: string, // ej. "10.00" (legible para humanos)
    chain: string = "polygon"
) {
    try {
        // Obtener el wallet usando el locator de email (formato: email:email@domain:chain)
        const walletLocator = `email:${senderEmail}:${chain}`;
        const wallet = await crossmintWallets.getWallet(walletLocator);

        // Envolver para operaciones EVM (SDK maneja automáticamente aprobaciones de signer, ej. enlace mágico por email)
        const evmWallet = EVMWallet.from(wallet);

        // Para transferencia nativa: Establecer value en hex wei (ej. ethers.parseEther(amount)), data: "0x", to: recipient
        // Para ERC-20 (ej. USDC en Polygon):
        const USDC_ADDRESS = "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"; // USDC en Polygon
        const ERC20_ABI = ["function transfer(address to, uint256 amount)"];
        const iface = new ethers.Interface(ERC20_ABI);
        const amountWei = ethers.parseUnits(amount, 6); // USDC: 6 decimales
        const data = iface.encodeFunctionData("transfer", [recipientAddress, amountWei]);

        // Enviar transacción
        const { hash, explorerLink } = await evmWallet.sendTransaction({
            to: USDC_ADDRESS,     // Contrato para tokens; destinatario para nativo
            value: "0x0",         // Hex wei para nativo; "0x0" para tokens
            data                  // "0x" para envíos simples/nativos
        });

        console.log("✅ Transacción enviada!");
        console.log("Hash:", hash);
        console.log("Explorer:", explorerLink);

        return { hash, explorerLink };
    } catch (error: any) {
        console.error("❌ Error enviando transacción:", error);
        if (error.message.includes("approval")) {
            console.log("Revisa el email para el enlace mágico de aprobación.");
        }
        throw error;
    }
}
```

**Ejemplo de Uso:**
```typescript
const result = await sendStablecoinTransfer(
    "sender@globosend.com",
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "10.00"  // $10 USDC
);
```

### Flujo Completo
```typescript
async function main() {
    // Paso 1: Crear wallet para el remitente
    const senderEmail = "sender@globosend.com";
    const chain = "polygon";
    const wallet = await createUserWallet(senderEmail, chain);

    // Paso 2: Enviar transferencia al destinatario (asegúrate de que el wallet esté financiado para fees/gas)
    const recipientAddress = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
    const result = await sendStablecoinTransfer(
        senderEmail,
        recipientAddress,
        "10.00",  // $10 USDC
        chain
    );

    console.log("✅ Transferencia completa:", result);
}

main().catch(console.error);
```

## Configuración

### Configuración de Clave API

1. Ve a [Crossmint Console](https://www.crossmint.com/console)
2. Navega a **Settings** → **API Keys**
3. Crea una nueva clave API del lado del servidor (nunca la expongas en el cliente)
4. Asegúrate de que tenga los scopes requeridos:
   - `wallets.create`
   - `wallets:transactions.create`

**Staging vs Producción:** Usa `environment: "staging"` en `createCrossmint` para pruebas (URL base: staging.crossmint.com). Cambia a producción para mainnet.

### Formato de Locator de Wallet

Al recuperar un wallet existente, usa identificadores flexibles:
```text
email:{user-email}:{chain-type}  // Específico de cadena
// O: email:{user-email}:evm     // Para cualquier cadena EVM
// O: {owner-type}:{value}:{chain} (ej. userId:123:polygon)
```

**Ejemplos:**

- `email:user@example.com:polygon`
- `email:user@example.com:base`
- `email:user@example.com:ethereum-sepolia`
- Dirección directa: `0x...` (si se conoce)

### Cadenas Soportadas

Crossmint soporta Smart Wallets en más de 50 cadenas. A continuación, un subconjunto relevante para wallets EVM (la lista completa incluye no-EVM como Solana). ✅ significa soportado para Smart Wallets; ✱ significa disponible bajo solicitud (contacta ventas).

| Blockchain     | Identificador    | Etiqueta Mainnet | Etiqueta Testnet    | Smart Wallets | Mejor Para        | Fees    | Notas                  |
|----------------|------------------|------------------|---------------------|---------------|-------------------|---------|------------------------|
| Polygon       | polygon         | `polygon`       | `polygon-amoy`     | ✅            | Stablecoins      | Baja   | Recomendado para transferencias |
| Base          | base            | `base`          | `base-sepolia`     | ✅            | Usuarios Coinbase| Baja   | Mainnet soportado      |
| Ethereum Sepolia | ethereum-sepolia | -             | `ethereum-sepolia` | ✅            | Pruebas          | Testnet| Predeterminado en staging |
| Arbitrum One  | arbitrum        | `arbitrum`      | `arbitrum-sepolia` | ✅            | Capa 2           | Baja   |                        |
| Optimism      | optimism        | `optimism`      | `optimism-sepolia` | ✅            | Capa 2           | Baja   |                        |
| Solana        | solana          | `solana`        | `solana`           | ✅            | Alto rendimiento | Variable| Ejemplo no-EVM        |
| ... (50+ más)| Varia           | Varia           | Varia              | Varia         | Varia            | Varia  | Ver tabla completa abajo  |

> **Nota:** En staging, solo se soportan cadenas testnet (ej. polygon-amoy, no polygon mainnet). Para cadenas ✱ o personalizadas, [contacta ventas](https://www.crossmint.com/contact/sales). Las cadenas EVM comparten claves.

[Tabla Completa de Cadenas Soportadas →](https://docs.crossmint.com/introduction/supported-chains) (incluye MPC Wallets, Minting, etc.)

### Parámetros de Transacción

Según EVMTransactionInput:

| Parámetro | Tipo | Descripción | Ejemplo |
|-----------|------|-------------|---------|
| `to` | string | Dirección de destinatario o contrato | `"0x742d35..."` |
| `value` | hex string | Cantidad de moneda nativa en wei (hex) | `"0x0"` (0) o `"0xde0b6b3a7640000"` (1 unidad) |
| `data` | hex string | Calldata codificada para contratos | `"0x"` (ninguno) o transferencia ERC-20 codificada |

**Aprobaciones de Signer:** El SDK las maneja automáticamente (ej. enlace mágico por email). Espera un email de aprobación para transacciones.

## Solución de Problemas

### Errores Comunes

#### ❌ `walletLocator is not defined` o Locator Inválido
**Solución:** Usa formato correcto (sensible a mayúsculas). Ejemplos:
```typescript
const walletLocator = `email:${userEmail}:polygon`; // O :evm para EVM
```

#### ❌ `Invalid API key` o Errores de Scope
**Soluciones:**
- Usa solo clave API del lado del servidor
- Verifica scopes en Console (`wallets.create`, `wallets:transactions.create`)
- Revisa mismatch de clave staging/prod

#### ❌ `Chain not supported`
**Soluciones:**
- Identificador exacto de la tabla (ej. "polygon", no personalizado)
- Testnets solo en staging (ej. "polygon-amoy")
- Revisa [cadenas soportadas](https://docs.crossmint.com/introduction/supported-chains); contacta para ✱

#### ❌ `Wallet already exists`
**Solución:** Usa `getWallet()` para recuperar; createWallet dará error si duplicado

#### ❌ Transacción Pendiente o Aprobación Fallida
**Solución:**
- Revisa spam para enlace mágico de signer por email
- Asegúrate de que el wallet tenga fondos para gas/fees
- Sondea estado de tx vía API si es necesario

### Modo Debug

Habilita logging detallado:
```typescript
const crossmint = createCrossmint({
    apiKey: "your-api-key",
    debug: true,  // Habilita logs de debug
});
```

## Recursos

- 📖 [Documentación Completa](https://docs.crossmint.com/wallets)
- 🔧 [Referencia API](https://docs.crossmint.com/api-reference)
- 💬 [Comunidad Discord](https://discord.gg/crossmint)
- 📧 [Soporte](mailto:support@crossmint.com)
- [Signers & Custody](/wallets/signers-and-custody) para auth avanzada
- [Tabla Completa de Cadenas Soportadas](https://docs.crossmint.com/introduction/supported-chains)

## Pasos Siguientes

- [ ] Agrega manejo de errores adecuado para producción (ej. reintento en aprobación pendiente)
- [ ] Implementa lógica de reintento para transacciones fallidas
- [ ] Agrega sondeo de estado de transacciones (vía API)
- [ ] Almacena direcciones/locators de wallets en tu base de datos
- [ ] Configura listeners de webhooks para eventos de transacciones
- [ ] Integra Crossmint Auth para resolución seamless de owner
- [ ] Prueba en staging con testnets (ej. polygon-amoy)

## Licencia

Este código de ejemplo se proporciona tal cual para fines de integración.

---

**¿Necesitas ayuda?** Abre un issue o contacta support@crossmint.com
