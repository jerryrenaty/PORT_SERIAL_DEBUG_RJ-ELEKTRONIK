# ⚡ Pilote de Débogage Universel Série (USB VCP / UART)

**Auteur :** RENATYJERRY  
**Date de création :** 15 mai 2026  
**Version :** 1.2  
**Cible :** Famille STM32 (Optimisé pour STM32H7)

---

## 📌 Présentation

Cette bibliothèque fournit une couche d'abstraction matérielle unifiée pour le débogage de vos applications électroniques **RJ-ELEKTRONIK**.  
Grâce à un unique sélecteur de configuration (`#define`), elle permet de basculer l'intégralité des flux d'envoi et de réception de données entre le port **USB virtuel (VCP - High Speed)** et un **UART matériel** (`huart1`, `huart2`, etc.), tout en conservant exactement les mêmes signatures de fonctions dans votre code applicatif.

### ✨ Fonctionnalités clés :
- **Bascule instantanée** USB <-> UART par configuration logicielle.
- **Tampon circulaire matériel autonome** de 512 octets pour la réception asynchrone sans blocage.
- **Modification dynamique du Baudrate** lors de l'initialisation du canal UART.
- **Sécurité d'envoi** avec gestion automatique du bus occupé (`USBD_BUSY`) et protection par Timeout (50ms).
- **Protection par section critique** (`__disable_irq`) lors du nettoyage du tampon.

---

## 🛠️ Configuration & Installation

### 1. Sélection de l'interface de communication
Ouvrez le fichier `PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.h` et configurez la macro `DEBUG_INTERFACE_SELECT` selon votre besoin actuel :

```c
// Pour utiliser le port USB Virtuel (VCP High-Speed) :
#define DEBUG_INTERFACE_SELECT   DEBUG_CHANNEL_USB

// OU pour utiliser un UART physique :
#define DEBUG_INTERFACE_SELECT   DEBUG_CHANNEL_UART
```

### 2. Routage des interruptions matérielles (Requis pour la réception)

#### 🔹 Cas du mode USB (`DEBUG_CHANNEL_USB`) :
Ouvrez le fichier généré par STM32CubeMX **`usbd_cdc_if.c`**, recherchez la fonction `CDC_Receive_HS` et insérez le code suivant entre les balises utilisateur :

```c
static int8_t CDC_Receive_HS(uint8_t* pbuf, uint32_t *Len)
{
  /* USER CODE BEGIN 11 */
  extern void Serial_Receive_Callback(uint8_t *Buf, uint32_t len);
  Serial_Receive_Callback(pbuf, *Len); // Injection dans le tampon circulaire
  
  USBD_CDC_SetRxBuffer(&hUsbDeviceHS, pbuf);
  USBD_CDC_ReceivePacket(&hUsbDeviceHS);
  return (USBD_OK);
  /* USER CODE END 11 */
}
```

#### 🔹 Cas du mode UART (`DEBUG_CHANNEL_UART`) :
Assurez-vous d'activer les interruptions globales de l'UART utilisé dans l'interface graphique de **STM32CubeMX** (*NVIC Settings -> USARTx global interrupt*). Le pilote capture automatiquement les octets via la fonction de rappel native `HAL_UART_RxCpltCallback`.

---

## 📖 Guide de Référence de l'API

### 1. Fonctions d'Initialisation

#### `void Serial_debug_RJ(void);` *(En mode USB)*
#### `void Serial_debug_RJ(UART_HandleTypeDef *huart, uint32_t baudrate);` *(En mode UART)*
Initialise les pointeurs, nettoie le tampon de réception circulaire et configure dynamiquement les registres matériels du microcontrôleur.
- **`huart`** : Pointeur vers la structure de configuration CubeMX de l'UART (ex: `&huart1`).
- **`baudrate`** : Vitesse en bauds de la liaison série (ex: `115200`, `921600`).

---

### 2. Fonctions d'Écriture (Émission)

#### `int16_t debug_print(const char *str);`
Envoie une chaîne de caractères textuelle classique sur le canal de débogage actif.
- **`str`** : Chaîne de texte (terminée par `\0`).
- **Retour** : `0` en cas de succès, valeur négative en cas d'erreur.

#### `int16_t Serial_WriteData(const uint8_t *pdata, uint16_t size);`
Envoie un bloc de données binaires brutes. Cette fonction est protégée contre le blocage du bus.
- **`pdata`** : Pointeur vers le tableau d'octets à transmettre.
- **`size`** : Nombre d'octets à envoyer.
- **Retour** : `0` = OK, `-1` = Paramètres invalides, `-2` = Timeout, `-3`/`-4` = Erreurs matérielles.

---

### 3. Fonctions de Lecture (Réception)

#### `uint16_t Serial_GetRxDataSize(void);`
Vérifie de manière non bloquante l'état du tampon circulaire.
- **Retour** : Le nombre d'octets actuellement disponibles et en attente de lecture.

#### `uint16_t Serial_ReadData(uint8_t *pdest, uint16_t max_size);`
Extrait les octets en attente du tampon de réception et les copie dans la mémoire de votre application.
- **`pdest`** : Pointeur vers votre tableau local de destination.
- **`max_size`** : Capacité maximale de stockage de votre tableau local.
- **Retour** : Le nombre exact d'octets qui ont été lus et transférés.

#### `void Serial_ClearRxBuffer(void);`
Vide instantanément l'intégralité du tampon de réception. Cette opération est protégée contre les interruptions concurrentes.

---

## 💻 Exemple d'Intégration Complet dans `main.c`

Voici comment intégrer de manière interactive et robuste le débogage dans le corps principal de votre application :

```c
#include "PORT_SERIAL_DEBUG_RJ-ELEKTRONIK.h"
#include <stdio.h>

int main(void)
{
  /* ... Initialisations système STM32 (HAL_Init, Clocks, GPIO...) ... */
  /* ... Initialisations de huart1 ou de l'USB générées par CubeMX ... */

  /* USER CODE BEGIN 2 */
  
  // 1. Initialisation unique adaptée à la compilation choisie
  #if (DEBUG_INTERFACE_SELECT == DEBUG_CHANNEL_USB)
    Serial_debug_RJ();       // Valeur de baudrate ignorée de façon transparente
  #else
    Serial_debug_RJ(&huart1, 921600); // Reconfigure l'UART1 à 921600 bauds à la volée
  #endif

  debug_print("\r\n=======================================\r\n");
  debug_print(" Système de Debug RJ-ELEKTRONIK Prêt !\r\n");
  debug_print("=======================================\r\n");

  /* USER CODE END 2 */

  uint8_t receive_byte;
  char msg_log[64];

  /* Boucle principale */
  while (1)
  {
    // 2. Lecture asynchrone non bloquante des commandes envoyées par le PC (ex: Web Serial / Termite)
    if (Serial_GetRxDataSize() > 0)
    {
      uint16_t read_bytes = Serial_ReadData(&receive_byte, 1);
      
      if (read_bytes > 0)
      {
        // 3. Traitement de l'octet et renvoi de confirmation en écho
        sprintf(msg_log, " -> Octet recu sur STM32 : '%c' (0x%02X)\r\n", receive_byte, receive_byte);
        debug_print(msg_log);
      }
    }

    /* Reste de la logique applicative (Mise à jour Flash, Écran LTDC...) */
  }
}
```
