markdown
# PORT_SERIAL_DEBUG_RJ-ELEKTRONIK

Une bibliothèque de débogage série légère et portable pour microcontrôleurs STM32, supportant USB (VCP) et UART avec une API unifiée.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![STM32](https://img.shields.io/badge/STM32-32F745-03234B.svg?logo=stmicroelectronics)](https://www.st.com)
[![Platform](https://img.shields.io/badge/platform-STM32-blue.svg)]()

## 📋 Table des matières

- [Fonctionnalités](#-fonctionnalités)
- [Architecture](#-architecture)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [API Reference](#-api-reference)
- [Exemples d'utilisation](#-exemples-dutilisation)
- [Cas d'usage avancés](#-cas-dusage-avancés)
- [Dépannage](#-dépannage)
- [Contribuer](#-contribuer)

## ✨ Fonctionnalités

- **Interface unique** : La même API fonctionne avec USB ou UART
- **Double interface** : Support du CDC Virtual COM Port (USB) et UART standard
- **Tampon circulaire** : Buffer de réception 512 octets sans perte
- **Mode non-bloquant** : Callback pour la réception de données
- **Configurable** : Vitesse USB HS/FS, baudrate UART modifiable dynamiquement
- **Portable** : Compatible avec tous les STM32 ayant USB et/ou UART
- **Printf-like** : Fonction `debug_println()` similaire à printf
- **Thread-safe** : Protection partielle des sections critiques

## 🏗 Architecture
┌─────────────────────────────────────────────────────────┐
│ APPLICATION LAYER │
│ debug_print() | debug_println() | Serial_ReadData() │
└────────────────────┬────────────────────────────────────┘
│
┌────────────────────▼────────────────────────────────────┐
│ UNIFIED API LAYER │
│ Serial_WriteData() | Serial_Receive_Callback() │
└────────────┬───────────────────────────┬────────────────┘
│ │
┌────────▼────────┐ ┌────────▼────────┐
│ USB VCP │ │ UART │
│ CDC_Transmit │ │ HAL_UART_ │
│ CDC_Receive │ │ Transmit/Receive│
└─────────────────┘ └─────────────────┘

text

## 📦 Installation

### 1. Prérequis

- Environnement STM32CubeIDE ou Makefile avec ARM GCC
- HAL Drivers STM32 générés par CubeMX
- Pour l'USB : Activation de USB_DEVICE avec classe CDC

### 2. Copie des fichiers

```bash
# Copier les fichiers dans votre projet
cp PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.h inc/
cp PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.c src/
3. Configuration CubeMX
Pour l'interface USB :
yaml
Middleware:
  USB_DEVICE:
    Class for FS IP: Communication Device Class (Virtual Port Com)
Pour l'interface UART :
yaml
Connectivity:
  USARTx:
    Mode: Asynchronous
    Hardware Flow Control: Disable
4. Inclusion dans main.c
c
#include "PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.h"

int main(void) {
    HAL_Init();
    SystemClock_Config();
    
    // Initialisation selon l'interface choisie
    #if (DEBUG_INTERFACE_SELECT == DEBUG_CHANNEL_USB)
        Serial_debug_RJ();
    #else
        extern UART_HandleTypeDef huart2;
        Serial_debug_RJ(&huart2, 115200);
    #endif
    
    // Votre code...
}
⚙️ Configuration
Configuration du mode de débogage
Dans PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.h :

c
// Choisir l'interface
#define DEBUG_INTERFACE_SELECT   DEBUG_CHANNEL_USB  // ou DEBUG_CHANNEL_UART

// Pour l'USB : choisir la vitesse
#define USB_SPEED_HIGH           1  // 1 = High Speed, 0 = Full Speed

// Pour l'UART : configurer dans l'initialisation
// Serial_debug_RJ(&huart2, 115200);
Paramètres ajustables
Paramètre	Valeur défaut	Description
VCP_RX_BUFFER_SIZE	512	Taille du buffer circulaire RX
Timeout envoi USB	50ms	Temps max d'attente transmission
Timeout envoi UART	100ms	Temps max d'attente transmission
📚 API Reference
Initialisation
c
// Pour USB
void Serial_debug_RJ(void);

// Pour UART
void Serial_debug_RJ(UART_HandleTypeDef *huart, uint32_t baudrate);
Envoi de données
c
// Envoyer une chaîne simple
int16_t debug_print(const char *str);

// Envoyer avec formatage (style printf)
int16_t debug_println(const char *format, ...);

// Envoyer des données binaires
int16_t Serial_WriteData(const uint8_t *pdata, uint16_t size);
Valeurs de retour :

0 : Succès

-1 : Paramètre invalide (NULL)

-2 : Timeout USB

-3 : Erreur USB générique

-4 : Erreur UART matérielle

Réception de données
c
// Obtenir le nombre d'octets disponibles
uint16_t Serial_GetRxDataSize(void);

// Lire les données reçues
uint16_t Serial_ReadData(uint8_t *pdest, uint16_t max_size);

// Vider le buffer de réception
void Serial_ClearRxBuffer(void);
Callback utilisateur
c
// À implémenter dans votre code si besoin de réception
void Serial_Receive_Callback(uint8_t *Buf, uint32_t len);
💡 Exemples d'utilisation
Exemple 1 : Débogage simple avec USB
c
#include "PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.h"

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_USB_DEVICE_Init();
    
    // Initialisation USB
    Serial_debug_RJ();
    
    // Messages de débogage
    debug_print("System starting...\n");
    debug_println("Value: %d, Hex: 0x%04X", 42, 0xABCD);
    
    uint32_t counter = 0;
    while (1) {
        debug_println("Loop iteration: %lu", counter++);
        HAL_Delay(1000);
    }
}
Exemple 2 : Configuration UART avec baudrate dynamique
c
#include "PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.h"

extern UART_HandleTypeDef huart2;

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_USART2_UART_Init();
    
    // Initialisation à 9600 bauds
    Serial_debug_RJ(&huart2, 9600);
    debug_println("Debug at 9600 bauds");
    
    // Changement de vitesse à 115200
    Serial_debug_RJ(&huart2, 115200);
    debug_println("Now at 115200 bauds");
    
    while (1) {
        // Vérifier les données reçues
        if (Serial_GetRxDataSize() > 0) {
            uint8_t buffer[64];
            uint16_t len = Serial_ReadData(buffer, sizeof(buffer)-1);
            buffer[len] = '\0';
            debug_println("Received: %s", buffer);
        }
        HAL_Delay(10);
    }
}
Exemple 3 : Réception de commandes
c
#define CMD_BUFFER_SIZE 64

static uint8_t cmd_buffer[CMD_BUFFER_SIZE];
static uint16_t cmd_index = 0;

// Callback appelé à chaque réception
void Serial_Receive_Callback(uint8_t *Buf, uint32_t len) {
    for (uint32_t i = 0; i < len; i++) {
        char c = Buf[i];
        
        if (c == '\r' || c == '\n') {
            // Commande terminée
            cmd_buffer[cmd_index] = '\0';
            process_command((char*)cmd_buffer);
            cmd_index = 0;
        } else if (cmd_index < CMD_BUFFER_SIZE - 1) {
            cmd_buffer[cmd_index++] = c;
        }
    }
}

void process_command(char *cmd) {
    if (strcmp(cmd, "status") == 0) {
        debug_println("System status: OK");
    } else if (strcmp(cmd, "reset") == 0) {
        debug_println("Resetting...");
        HAL_NVIC_SystemReset();
    } else {
        debug_println("Unknown command: %s", cmd);
    }
}

int main(void) {
    Serial_debug_RJ();  // Mode USB
    
    while (1) {
        // Le callback gère tout
        HAL_Delay(100);
    }
}
Exemple 4 : Affichage de données binaires (hexdump)
c
void hex_dump(uint8_t *data, uint16_t len) {
    for (uint16_t i = 0; i < len; i++) {
        debug_println("%02X ", data[i]);
        if ((i + 1) % 16 == 0) debug_print("\n");
    }
    debug_print("\n");
}

// Envoi de données binaires sans formatage
void send_binary_data(void) {
    uint8_t sensor_data[] = {0x01, 0x02, 0x03, 0xFF, 0x00};
    Serial_WriteData(sensor_data, sizeof(sensor_data));
}
Exemple 5 : Mode debug conditionnel
c
#define DEBUG_ENABLE 1

#if DEBUG_ENABLE
    #define DEBUG_PRINT(...) debug_println(__VA_ARGS__)
#else
    #define DEBUG_PRINT(...)
#endif

int main(void) {
    Serial_debug_RJ();
    
    DEBUG_PRINT("Program started");
    DEBUG_PRINT("Temperature: %.2f°C", 23.5);
    
    // Désactiver le debug en production
}
🔧 Cas d'usage avancés
Cas 1 : Double interface (USB + UART)
c
// Modifier le fichier .h pour ajouter un deuxième canal
#define DEBUG_INTERFACE_SELECT   DEBUG_CHANNEL_BOTH

// Implémentation possible :
void debug_print_both(const char *str) {
    #ifdef USB_TRANSMIT_FUNC
        USB_TRANSMIT_FUNC((uint8_t*)str, strlen(str));
    #endif
    if (static_huart) {
        HAL_UART_Transmit(static_huart, (uint8_t*)str, strlen(str), 100);
    }
}
Cas 2 : Logger avec timestamp
c
uint32_t get_ms_timestamp(void) {
    return HAL_GetTick();
}

void debug_log(const char *level, const char *format, ...) {
    char timestamp_buffer[32];
    char message_buffer[256];
    va_list args;
    
    snprintf(timestamp_buffer, sizeof(timestamp_buffer), "[%lu][%s] ", 
             get_ms_timestamp(), level);
    
    va_start(args, format);
    vsnprintf(message_buffer, sizeof(message_buffer), format, args);
    va_end(args);
    
    debug_print(timestamp_buffer);
    debug_println("%s", message_buffer);
}

// Utilisation
debug_log("INFO", "System initialized");
debug_log("ERROR", "Sensor %d failed", sensor_id);
Cas 3 : Buffer circulaire externe (DMA)
c
// Pour UART avec DMA - modifications suggérées
#define DMA_BUFFER_SIZE 1024
static uint8_t dma_buffer[DMA_BUFFER_SIZE];

void Serial_Init_DMA(UART_HandleTypeDef *huart) {
    // Configuration DMA circulaire
    HAL_UART_Receive_DMA(huart, dma_buffer, DMA_BUFFER_SIZE);
    
    // Utiliser le callback DMA pour remplir le buffer circulaire
}

void HAL_UART_RxHalfCpltCallback(UART_HandleTypeDef *huart) {
    // Traitement de la première moitié du buffer
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    // Traitement de la deuxième moitié
}
🐛 Dépannage
Problème 1 : Rien ne s'affiche sur le terminal
Solutions :

Vérifier la configuration USB/UART

Pour USB : Vérifier que CDC_Transmit_HS/FS est bien appelée

Pour UART : Vérifier les connexions physiques (TX/RX/GND)

Vérifier la vitesse de baudrate (doit correspondre)

Problème 2 : Perte de données en réception
Causes possibles :

Buffer trop petit : Augmenter VCP_RX_BUFFER_SIZE

Temps de traitement trop long : Optimiser le callback

Interruptions mal configurées

Solution :

c
// Augmenter le buffer
#define VCP_RX_BUFFER_SIZE 2048

// Dans le callback, être rapide
void Serial_Receive_Callback(uint8_t *Buf, uint32_t len) {
    // Copie rapide, pas de traitement lourd ici
    for (uint32_t i = 0; i < len; i++) {
        // Simple copie dans le buffer circulaire
        // Pas de printf, pas de traitement
    }
}
Problème 3 : Freeze lors de l'envoi USB
Solution : Vérifier que vous n'appelez pas debug_print() dans une interruption prioritaire. Le timeout est normalement géré.

Problème 4 : Erreur de compilation "usbd_cdc_if.h not found"
Solution : Dans CubeMX, activer correctement l'USB avec la classe CDC :

Middleware → USB_DEVICE → Class for FS IP : "Communication Device Class (VCP)"

Générer le code

📊 Performance
Interface	Vitesse max théorique	Latence minimale	Buffer recommandé
USB HS	480 Mbps	~1ms	512 - 2048 bytes
USB FS	12 Mbps	~1ms	512 - 1024 bytes
UART	Jusqu'à 921600 bauds	Variable	256 - 512 bytes
🤝 Contribuer
Les contributions sont les bienvenues !

Forkez le projet

Créez votre branche (git checkout -b feature/AmazingFeature)

Committez (git commit -m 'Add some AmazingFeature')

Pushez (git push origin feature/AmazingFeature)

Ouvrez une Pull Request

Rapports de bugs
Utilisez les issues GitHub avec :

Version de STM32

Configuration (USB/UART)

Code minimal reproduisant le problème

Comportement attendu vs réel

📝 License
Distribué sous la licence MIT. Voir LICENSE pour plus d'informations.

👤 Auteur
RENATYJERRY - Travail initial - GitHub

🙏 Remerciements
Équipe STMicroelectronics pour les HAL Drivers

Communauté STM32 pour les retours et suggestions

⭐ N'oubliez pas de mettre une étoile si ce projet vous a été utile !

text

Ce README complet inclut :
- Une documentation professionnelle et structurée
- Des exemples pour tous les cas d'utilisation
- Une section dépannage détaillée
- Des informations de performance
- Des instructions pour la contribution
- Un format adapté pour GitHub (avec badges, tables, code blocks)
