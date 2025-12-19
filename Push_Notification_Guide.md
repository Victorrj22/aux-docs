# Guia Completo de Implementacao de Push Notifications

## Expo (React Native) + .NET Backend + Firebase Cloud Messaging

Este guia documenta o processo completo para implementar push notifications em um app React Native com Expo e backend .NET, usando Firebase Cloud Messaging (FCM) via Expo Push Service.

---

## Arquitetura Geral

```
+------------------+     +-------------------+     +------------------+
|   React Native   |     |    .NET Backend   |     |  Expo Push API   |
|   (Expo)         |     |                   |     |  + Firebase FCM  |
+------------------+     +-------------------+     +------------------+
        |                        |                        |
        | 1. Obtem Expo Token    |                        |
        |----------------------->| 2. Armazena Token      |
        |                        |                        |
        |                        | 3. Envia Push -------->|
        |                        |    (via Expo API)      |
        |                        |                        |
        | 4. Recebe Push <--------------------------------|
        |                        |                        |
+------------------+     +-------------------+     +------------------+
```

### Fluxo Resumido:
1. App obtem o **Expo Push Token** do dispositivo
2. App envia o token para o backend, que armazena no banco
3. Quando precisa notificar, backend envia para **Expo Push API**
4. Expo Push API roteia para Firebase (Android) ou APNs (iOS)
5. Dispositivo recebe a notificacao

---

## PARTE 1: Configuracao do Firebase

### 1.1 Criar Projeto Firebase

1. Acesse [Firebase Console](https://console.firebase.google.com/)
2. Clique em **"Adicionar projeto"**
3. De um nome ao projeto (ex: "MeuApp")
4. Desabilite Google Analytics se nao precisar
5. Clique em **"Criar projeto"**

### 1.2 Adicionar App Android

1. No painel do projeto, clique no icone do Android
2. Preencha o **Package name** (ex: `com.empresa.meuapp`)
   - IMPORTANTE: Deve ser igual ao `package` em `app.json`
3. Apelido do app: nome de exibicao
4. Clique em **"Registrar app"**
5. Baixe o arquivo `google-services.json`
6. Coloque o arquivo na raiz do projeto Expo

### 1.3 Adicionar App iOS (se aplicavel)

1. No painel do projeto, clique no icone da Apple
2. Preencha o **Bundle ID** (ex: `com.empresa.meuapp`)
   - IMPORTANTE: Deve ser igual ao `bundleIdentifier` em `app.json`
3. Clique em **"Registrar app"**
4. Baixe o arquivo `GoogleService-Info.plist`
5. Coloque o arquivo na raiz do projeto Expo

### 1.4 Configurar FCM V1 (Service Account Key)

**IMPORTANTE**: O Firebase descontinuou a "Server Key" legada. Use FCM V1 com Service Account.

1. No Firebase Console, va para **Configuracoes do Projeto** (icone de engrenagem)
2. Aba **"Contas de servico"**
3. Clique em **"Gerar nova chave privada"**
4. Baixe o arquivo JSON (ex: `firebase-service-account.json`)

### 1.5 Configurar Credenciais no Expo

1. Acesse [expo.dev](https://expo.dev)
2. Va para seu projeto > **Credentials**
3. Selecione **Android** > **FCM V1 service account key**
4. Clique em **"Add a service account key"**
5. Faca upload do JSON baixado no passo anterior

Para iOS, configure tambem o APNs Key nas credenciais do Expo.

---

## PARTE 2: Configuracao do Projeto Expo

### 2.1 Instalar Dependencias

```bash
npx expo install expo-notifications expo-device expo-constants
```

### 2.2 Configurar app.json

```json
{
  "expo": {
    "name": "MeuApp",
    "slug": "meuapp",
    "owner": "seu-usuario-expo",
    "scheme": "meuapp",
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.empresa.meuapp",
      "infoPlist": {
        "UIBackgroundModes": ["remote-notification"]
      }
    },
    "android": {
      "package": "com.empresa.meuapp",
      "googleServicesFile": "./google-services.json"
    },
    "plugins": [
      [
        "expo-notifications",
        {
          "icon": "./assets/notification-icon.png",
          "color": "#FF6B00"
        }
      ]
    ],
    "extra": {
      "eas": {
        "projectId": "SEU-PROJECT-ID-DO-EXPO"
      }
    }
  }
}
```

**IMPORTANTE**: O `projectId` e obtido em expo.dev > seu projeto > Project ID

### 2.3 Criar notificationService.ts

```typescript
// src/services/notificationService.ts
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';
import { Platform } from 'react-native';

// Tipos de notificacao (devem corresponder ao backend)
export const NotificationType = {
  NovaMensagem: 'nova_mensagem',
  NovoEvento: 'novo_evento',
  InscricaoAprovada: 'inscricao_aprovada',
  // ... adicione seus tipos
} as const;

export type NotificationTypeValue = (typeof NotificationType)[keyof typeof NotificationType];

// Verifica se esta rodando no Expo Go (limitado para push)
const isExpoGo = Constants.executionEnvironment === 'storeClient';

// Configura como notificacoes sao exibidas em foreground
if (!isExpoGo || Platform.OS === 'ios') {
  Notifications.setNotificationHandler({
    handleNotification: async () => ({
      shouldShowAlert: true,
      shouldPlaySound: true,
      shouldSetBadge: true,
    }),
  });
}

export type PushTokenResult = {
  token: string | null;
  error?: string;
};

export type NotificationData = {
  tipo?: NotificationTypeValue;
  feiraId?: number;
  conversaId?: number;
  // ... adicione seus campos
  [key: string]: unknown;
};

/**
 * Solicita permissao de notificacoes
 */
export async function requestNotificationPermissions(): Promise<boolean> {
  if (!Device.isDevice) {
    console.log('[Notifications] Simulador - push nao funciona');
    return false;
  }

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  return finalStatus === 'granted';
}

/**
 * Obtem o Expo Push Token do dispositivo
 */
export async function getExpoPushToken(): Promise<PushTokenResult> {
  // Push nao suportado no Expo Go Android desde SDK 53
  if (isExpoGo && Platform.OS === 'android') {
    return { token: null, error: 'Push nao suportado no Expo Go Android' };
  }

  if (!Device.isDevice) {
    return { token: null, error: 'Requer dispositivo fisico' };
  }

  const hasPermission = await requestNotificationPermissions();
  if (!hasPermission) {
    return { token: null, error: 'Permissao negada' };
  }

  // Obtem projectId de multiplas fontes para compatibilidade
  const projectId =
    Constants.expoConfig?.extra?.eas?.projectId ??
    Constants.manifest?.extra?.eas?.projectId ??
    'SEU-PROJECT-ID-FALLBACK';

  const tokenData = await Notifications.getExpoPushTokenAsync({ projectId });

  // Configura canais Android
  if (Platform.OS === 'android') {
    await setupAndroidChannels();
  }

  return { token: tokenData.data };
}

/**
 * Configura canais de notificacao Android
 */
async function setupAndroidChannels(): Promise<void> {
  await Notifications.setNotificationChannelAsync('default', {
    name: 'Geral',
    importance: Notifications.AndroidImportance.HIGH,
    vibrationPattern: [0, 250, 250, 250],
    lightColor: '#FF6B00',
  });

  await Notifications.setNotificationChannelAsync('messages', {
    name: 'Mensagens',
    importance: Notifications.AndroidImportance.HIGH,
    vibrationPattern: [0, 250, 250, 250],
  });

  // Adicione mais canais conforme necessario
}

/**
 * Listener para notificacoes recebidas (app em foreground)
 */
export function addNotificationReceivedListener(
  callback: (notification: Notifications.Notification) => void
): Notifications.EventSubscription {
  return Notifications.addNotificationReceivedListener(callback);
}

/**
 * Listener para quando usuario toca na notificacao
 */
export function addNotificationResponseReceivedListener(
  callback: (response: Notifications.NotificationResponse) => void
): Notifications.EventSubscription {
  return Notifications.addNotificationResponseReceivedListener(callback);
}

/**
 * Remove subscription de listener
 */
export function removeNotificationSubscription(
  subscription: Notifications.EventSubscription
): void {
  subscription.remove();
}

/**
 * Obtem ultima notificacao (se app foi aberto por ela)
 */
export async function getLastNotificationResponse(): Promise<Notifications.NotificationResponse | null> {
  return await Notifications.getLastNotificationResponseAsync();
}

/**
 * Define contador de badge no icone do app
 */
export async function setBadgeCount(count: number): Promise<void> {
  await Notifications.setBadgeCountAsync(count);
}

/**
 * Parseia dados da notificacao (campos podem vir como string ou number)
 */
function parseNumericField(value: unknown): number | undefined {
  if (typeof value === 'number') return value;
  if (typeof value === 'string') {
    const parsed = parseInt(value, 10);
    return isNaN(parsed) ? undefined : parsed;
  }
  return undefined;
}

export function parseNotificationData(
  response: Notifications.NotificationResponse
): NotificationData {
  const data = response.notification.request.content.data as Record<string, unknown>;
  return {
    tipo: data?.tipo as NotificationTypeValue | undefined,
    feiraId: parseNumericField(data?.feiraId ?? data?.feira_id),
    conversaId: parseNumericField(data?.conversaId ?? data?.conversa_id),
    // Adicione mais campos conforme necessario
    ...data,
  };
}
```

### 2.4 Criar NotificationContext.tsx

```typescript
// src/context/NotificationContext.tsx
import React, { createContext, useContext, useEffect, useState, useCallback, useRef } from 'react';
import * as Notifications from 'expo-notifications';
import { useNavigation, NavigationProp } from '@react-navigation/native';
import { AppState, AppStateStatus, Platform } from 'react-native';

import {
  getExpoPushToken,
  addNotificationReceivedListener,
  addNotificationResponseReceivedListener,
  removeNotificationSubscription,
  getLastNotificationResponse,
  setBadgeCount,
  parseNotificationData,
  NotificationData,
  NotificationType,
} from '../services/notificationService';
import { useAuth } from './AuthContext';
import { useApi } from '../services/api';

type NotificationContextValue = {
  expoPushToken: string | null;
  unreadCount: number;
  refreshUnreadCount: () => Promise<void>;
  isRegistered: boolean;
};

const NotificationContext = createContext<NotificationContextValue | undefined>(undefined);

// Defina os tipos de navegacao do seu app
type RootStackParamList = {
  MainTabs: { screen?: string };
  Chat: { conversaId: number; outroParticipante: { id: number; nome: string } };
  // ... adicione suas telas
};

export function NotificationProvider({ children }: { children: React.ReactNode }) {
  const { user, isAuthenticated } = useAuth();
  const api = useApi();
  const navigation = useNavigation<NavigationProp<RootStackParamList>>();

  const [expoPushToken, setExpoPushToken] = useState<string | null>(null);
  const [unreadCount, setUnreadCount] = useState(0);
  const [isRegistered, setIsRegistered] = useState(false);

  const notificationListener = useRef<Notifications.EventSubscription | null>(null);
  const responseListener = useRef<Notifications.EventSubscription | null>(null);
  const appState = useRef<AppStateStatus>(AppState.currentState);

  // Busca contagem de nao lidas da API
  const refreshUnreadCount = useCallback(async () => {
    if (!user?.id) {
      setUnreadCount(0);
      return;
    }
    try {
      const count = await api.contarNotificacoesNaoLidas(user.id);
      setUnreadCount(count);
      await setBadgeCount(count);
    } catch (error) {
      console.error('[NotificationContext] Erro ao buscar contagem:', error);
    }
  }, [user?.id, api]);

  // Registra token no backend
  const registerPushToken = useCallback(async (token: string) => {
    if (!user?.id || !token) return;

    try {
      const platform = Platform.OS === 'ios' ? 'ios' : 'android';
      await fetch(`${API_URL}/api/device-tokens/register`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          clienteId: user.id,
          token,
          platform,
        }),
      });
      setIsRegistered(true);
    } catch (error) {
      console.error('[NotificationContext] Erro ao registrar token:', error);
    }
  }, [user?.id]);

  // Desregistra token (logout)
  const unregisterPushToken = useCallback(async () => {
    if (!user?.id || !expoPushToken) return;

    try {
      await fetch(`${API_URL}/api/device-tokens/unregister`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          clienteId: user.id,
          token: expoPushToken,
        }),
      });
      setIsRegistered(false);
    } catch (error) {
      console.error('[NotificationContext] Erro ao desregistrar token:', error);
    }
  }, [user?.id, expoPushToken]);

  // Navega baseado no tipo de notificacao
  const handleNotificationNavigation = useCallback(async (data: NotificationData) => {
    if (!data.tipo) return;

    switch (data.tipo) {
      case NotificationType.NovaMensagem:
        if (data.conversaId && user?.id) {
          // IMPORTANTE: Busque dados necessarios antes de navegar
          const conversas = await api.listarConversas(user.id);
          const conversa = conversas.find(c => c.id === data.conversaId);
          if (conversa) {
            navigation.navigate('Chat', {
              conversaId: data.conversaId,
              outroParticipante: conversa.outroParticipante,
            });
          }
        }
        break;
      // ... adicione mais casos
      default:
        navigation.navigate('MainTabs', { screen: 'Notifications' });
    }
  }, [navigation, user?.id, api]);

  // Inicializa push notifications
  useEffect(() => {
    if (!isAuthenticated || !user?.id) {
      setExpoPushToken(null);
      setIsRegistered(false);
      return;
    }

    let isMounted = true;

    const init = async () => {
      const result = await getExpoPushToken();
      if (isMounted && result.token) {
        setExpoPushToken(result.token);
        await registerPushToken(result.token);
      }
    };

    init();

    return () => { isMounted = false; };
  }, [isAuthenticated, user?.id, registerPushToken]);

  // Configura listeners
  useEffect(() => {
    // Notificacao recebida em foreground
    notificationListener.current = addNotificationReceivedListener(() => {
      refreshUnreadCount();
    });

    // Usuario tocou na notificacao
    responseListener.current = addNotificationResponseReceivedListener((response) => {
      const data = parseNotificationData(response);
      handleNotificationNavigation(data);
      refreshUnreadCount();
    });

    // Verifica se app foi aberto por notificacao
    getLastNotificationResponse().then((response) => {
      if (response) {
        const data = parseNotificationData(response);
        setTimeout(() => handleNotificationNavigation(data), 500);
      }
    });

    return () => {
      if (notificationListener.current) removeNotificationSubscription(notificationListener.current);
      if (responseListener.current) removeNotificationSubscription(responseListener.current);
    };
  }, [handleNotificationNavigation, refreshUnreadCount]);

  // Atualiza contagem quando app volta ao foreground
  useEffect(() => {
    const subscription = AppState.addEventListener('change', (nextAppState) => {
      if (appState.current.match(/inactive|background/) && nextAppState === 'active') {
        refreshUnreadCount();
      }
      appState.current = nextAppState;
    });

    return () => subscription.remove();
  }, [refreshUnreadCount]);

  // Carrega contagem inicial
  useEffect(() => {
    if (isAuthenticated && user?.id) {
      refreshUnreadCount();
    }
  }, [isAuthenticated, user?.id, refreshUnreadCount]);

  // Cleanup no logout
  useEffect(() => {
    if (!isAuthenticated && expoPushToken) {
      unregisterPushToken();
    }
  }, [isAuthenticated, expoPushToken, unregisterPushToken]);

  return (
    <NotificationContext.Provider value={{ expoPushToken, unreadCount, refreshUnreadCount, isRegistered }}>
      {children}
    </NotificationContext.Provider>
  );
}

export function useNotifications(): NotificationContextValue {
  const context = useContext(NotificationContext);
  if (!context) {
    throw new Error('useNotifications deve ser usado dentro de NotificationProvider');
  }
  return context;
}
```

### 2.5 Adicionar Provider no App

```typescript
// App.tsx ou AppRoot.tsx
import { NotificationProvider } from './src/context/NotificationContext';

export default function App() {
  return (
    <AuthProvider>
      <NavigationContainer>
        <NotificationProvider>
          {/* Resto do app */}
        </NotificationProvider>
      </NavigationContainer>
    </AuthProvider>
  );
}
```

---

## PARTE 3: Backend .NET

### 3.1 Modelo DeviceToken

```csharp
// Models/DeviceToken.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace MeuApp.Models;

[Table("device_tokens", Schema = "public")]
public class DeviceToken
{
    [Key]
    [Column("id")]
    public int Id { get; set; }

    [Column("cliente_id")]
    public int ClienteId { get; set; }

    [Column("token")]
    [Required]
    public string Token { get; set; } = "";

    [Column("platform")]
    [Required]
    [MaxLength(10)]
    public string Platform { get; set; } = ""; // "ios" ou "android"

    [Column("is_active")]
    public bool IsActive { get; set; } = true;

    [Column("criado_em")]
    public DateTime CriadoEm { get; set; } = DateTime.UtcNow;

    [Column("atualizado_em")]
    public DateTime? AtualizadoEm { get; set; }

    // Navigation property
    [ForeignKey("ClienteId")]
    public virtual Cliente? Cliente { get; set; }
}
```

### 3.2 Modelo Notificacao (opcional, para historico)

```csharp
// Models/Notificacao.cs
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace MeuApp.Models;

[Table("notificacoes", Schema = "public")]
public class Notificacao
{
    [Key]
    [Column("id")]
    public int Id { get; set; }

    [Column("cliente_id")]
    public int ClienteId { get; set; }

    [Column("tipo")]
    [Required]
    [MaxLength(50)]
    public string Tipo { get; set; } = "";

    [Column("titulo")]
    [Required]
    [MaxLength(200)]
    public string Titulo { get; set; } = "";

    [Column("mensagem")]
    [Required]
    public string Mensagem { get; set; } = "";

    [Column("lida")]
    public bool Lida { get; set; } = false;

    [Column("criado_em")]
    public DateTime CriadoEm { get; set; } = DateTime.UtcNow;

    [Column("dados_extras")]
    public string? DadosExtras { get; set; } // JSON com dados adicionais

    // Foreign keys opcionais para navegacao
    [Column("feira_id")]
    public int? FeiraId { get; set; }

    [Column("inscricao_id")]
    public int? InscricaoId { get; set; }

    [Column("remetente_id")]
    public int? RemetenteId { get; set; }

    // Navigation properties
    [ForeignKey("ClienteId")]
    public virtual Cliente? Cliente { get; set; }
}
```

### 3.3 Tipos de Notificacao

```csharp
// Models/TipoNotificacao.cs
namespace MeuApp.Models;

public static class TipoNotificacao
{
    public const string NovaMensagem = "nova_mensagem";
    public const string NovoEvento = "novo_evento";
    public const string EventoAprovado = "evento_aprovado";
    public const string InscricaoRecebida = "inscricao_recebida";
    public const string InscricaoAprovada = "inscricao_aprovada";
    public const string InscricaoRejeitada = "inscricao_rejeitada";
    public const string PagamentoRecebido = "pagamento_recebido";
    public const string LembretePagamento = "lembrete_pagamento";
    // ... adicione mais tipos
}
```

### 3.4 DbContext

```csharp
// ExternalData/AppDbContext.cs
public class AppDbContext : DbContext
{
    public DbSet<DeviceToken> DeviceTokens { get; set; }
    public DbSet<Notificacao> Notificacoes { get; set; }
    // ... outros DbSets

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Indice unico para evitar tokens duplicados
        modelBuilder.Entity<DeviceToken>()
            .HasIndex(dt => new { dt.ClienteId, dt.Token })
            .IsUnique();
    }
}
```

### 3.5 Migration SQL

```sql
-- Migration para criar tabelas

-- Tabela de device tokens
CREATE TABLE public.device_tokens (
    id SERIAL PRIMARY KEY,
    cliente_id INT NOT NULL REFERENCES public.clientes(id) ON DELETE CASCADE,
    token TEXT NOT NULL,
    platform VARCHAR(10) NOT NULL CHECK (platform IN ('ios', 'android')),
    is_active BOOLEAN DEFAULT true,
    criado_em TIMESTAMP DEFAULT NOW(),
    atualizado_em TIMESTAMP
);

CREATE UNIQUE INDEX idx_device_tokens_unique ON public.device_tokens(cliente_id, token);
CREATE INDEX idx_device_tokens_cliente ON public.device_tokens(cliente_id) WHERE is_active = true;

-- Tabela de notificacoes (historico)
CREATE TABLE public.notificacoes (
    id SERIAL PRIMARY KEY,
    cliente_id INT NOT NULL REFERENCES public.clientes(id) ON DELETE CASCADE,
    tipo VARCHAR(50) NOT NULL,
    titulo VARCHAR(200) NOT NULL,
    mensagem TEXT NOT NULL,
    lida BOOLEAN DEFAULT false,
    criado_em TIMESTAMP DEFAULT NOW(),
    dados_extras TEXT,
    feira_id INT REFERENCES public.feiras(id) ON DELETE SET NULL,
    inscricao_id INT REFERENCES public.inscricoes(id) ON DELETE SET NULL,
    remetente_id INT REFERENCES public.clientes(id) ON DELETE SET NULL
);

CREATE INDEX idx_notificacoes_cliente ON public.notificacoes(cliente_id);
CREATE INDEX idx_notificacoes_nao_lidas ON public.notificacoes(cliente_id) WHERE lida = false;
```

### 3.6 PushNotificationService

```csharp
// Services/PushNotificationService.cs
using System.Text;
using System.Text.Json;
using System.Text.Json.Serialization;
using Microsoft.EntityFrameworkCore;
using MeuApp.ExternalData;
using MeuApp.Models;

namespace MeuApp.Services;

public interface IPushNotificationService
{
    Task SendPushNotificationAsync(int clienteId, string titulo, string mensagem,
        Dictionary<string, object>? data = null, CancellationToken ct = default);

    Task SendPushNotificationToMultipleAsync(IEnumerable<int> clienteIds, string titulo,
        string mensagem, Dictionary<string, object>? data = null, CancellationToken ct = default);

    Task<Notificacao> CreateAndSendNotificationAsync(int clienteId, string tipo, string titulo,
        string mensagem, int? feiraId = null, int? inscricaoId = null, int? remetenteId = null,
        string? dadosExtras = null, CancellationToken ct = default);
}

public class PushNotificationService : IPushNotificationService
{
    private readonly AppDbContext _db;
    private readonly ILogger<PushNotificationService> _logger;
    private readonly HttpClient _httpClient;

    // Endpoint da Expo Push API
    private const string EXPO_PUSH_ENDPOINT = "https://exp.host/--/api/v2/push/send";

    public PushNotificationService(
        AppDbContext db,
        ILogger<PushNotificationService> logger,
        IHttpClientFactory httpClientFactory)
    {
        _db = db;
        _logger = logger;
        _httpClient = httpClientFactory.CreateClient("ExpoPush");
    }

    public async Task SendPushNotificationAsync(int clienteId, string titulo, string mensagem,
        Dictionary<string, object>? data = null, CancellationToken ct = default)
    {
        try
        {
            // Busca tokens ativos do cliente
            var tokens = await _db.DeviceTokens
                .Where(dt => dt.ClienteId == clienteId && dt.IsActive)
                .Select(dt => dt.Token)
                .ToListAsync(ct);

            if (tokens.Count == 0)
            {
                _logger.LogDebug("Nenhum token ativo para cliente {ClienteId}", clienteId);
                return;
            }

            await SendPushToTokensAsync(tokens, titulo, mensagem, data, ct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao enviar push para cliente {ClienteId}", clienteId);
        }
    }

    public async Task SendPushNotificationToMultipleAsync(IEnumerable<int> clienteIds,
        string titulo, string mensagem, Dictionary<string, object>? data = null,
        CancellationToken ct = default)
    {
        try
        {
            var clienteIdsList = clienteIds.ToList();
            var tokens = await _db.DeviceTokens
                .Where(dt => clienteIdsList.Contains(dt.ClienteId) && dt.IsActive)
                .Select(dt => dt.Token)
                .ToListAsync(ct);

            if (tokens.Count == 0) return;

            await SendPushToTokensAsync(tokens, titulo, mensagem, data, ct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao enviar push para multiplos clientes");
        }
    }

    public async Task<Notificacao> CreateAndSendNotificationAsync(int clienteId, string tipo,
        string titulo, string mensagem, int? feiraId = null, int? inscricaoId = null,
        int? remetenteId = null, string? dadosExtras = null, CancellationToken ct = default)
    {
        // Cria notificacao no banco (historico)
        var notificacao = new Notificacao
        {
            ClienteId = clienteId,
            Tipo = tipo,
            Titulo = titulo,
            Mensagem = mensagem,
            FeiraId = feiraId,
            InscricaoId = inscricaoId,
            RemetenteId = remetenteId,
            DadosExtras = dadosExtras,
            CriadoEm = DateTime.UtcNow,
            Lida = false
        };

        _db.Notificacoes.Add(notificacao);
        await _db.SaveChangesAsync(ct);

        // Monta dados do push
        var data = new Dictionary<string, object>
        {
            { "notificacao_id", notificacao.Id },
            { "tipo", tipo }
        };

        if (feiraId.HasValue) data["feira_id"] = feiraId.Value;
        if (inscricaoId.HasValue) data["inscricao_id"] = inscricaoId.Value;

        // Envia push
        await SendPushNotificationAsync(clienteId, titulo, mensagem, data, ct);

        return notificacao;
    }

    private async Task SendPushToTokensAsync(List<string> tokens, string titulo,
        string mensagem, Dictionary<string, object>? data, CancellationToken ct)
    {
        // Expo aceita ate 100 mensagens por request
        const int batchSize = 100;
        for (int i = 0; i < tokens.Count; i += batchSize)
        {
            var batch = tokens.Skip(i).Take(batchSize).ToList();
            await SendExpoBatchAsync(batch, titulo, mensagem, data, ct);
        }
    }

    private async Task SendExpoBatchAsync(List<string> tokens, string titulo,
        string mensagem, Dictionary<string, object>? data, CancellationToken ct)
    {
        try
        {
            // Monta array de mensagens
            var messages = tokens.Select(token => new ExpoPushMessage
            {
                To = token,
                Title = titulo,
                Body = mensagem,
                Sound = "default",
                Data = data,
                Priority = "high",
                ChannelId = "default"
            }).ToList();

            var json = JsonSerializer.Serialize(messages, new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
            });

            var content = new StringContent(json, Encoding.UTF8, "application/json");

            using var request = new HttpRequestMessage(HttpMethod.Post, EXPO_PUSH_ENDPOINT);
            request.Content = content;
            request.Headers.Accept.Add(
                new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/json"));

            var response = await _httpClient.SendAsync(request, ct);
            var responseBody = await response.Content.ReadAsStringAsync(ct);

            if (!response.IsSuccessStatusCode)
            {
                _logger.LogError("Expo Push falhou: {Status} - {Body}",
                    response.StatusCode, responseBody);
            }
            else
            {
                // Processa resposta para desativar tokens invalidos
                await ProcessExpoResponseAsync(tokens, responseBody, ct);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao enviar batch Expo Push");
        }
    }

    private async Task ProcessExpoResponseAsync(List<string> tokens, string responseBody,
        CancellationToken ct)
    {
        try
        {
            using var doc = JsonDocument.Parse(responseBody);
            var root = doc.RootElement;

            if (!root.TryGetProperty("data", out var dataArray)) return;

            var invalidTokens = new List<string>();
            var results = dataArray.EnumerateArray().ToList();

            for (int i = 0; i < results.Count && i < tokens.Count; i++)
            {
                var result = results[i];
                if (result.TryGetProperty("status", out var status) &&
                    status.GetString() == "error")
                {
                    if (result.TryGetProperty("details", out var details) &&
                        details.TryGetProperty("error", out var errorType))
                    {
                        var errorStr = errorType.GetString();
                        // Token invalido ou dispositivo desregistrado
                        if (errorStr == "DeviceNotRegistered" ||
                            errorStr == "InvalidCredentials")
                        {
                            invalidTokens.Add(tokens[i]);
                        }
                    }
                }
            }

            // Desativa tokens invalidos
            if (invalidTokens.Count > 0)
            {
                await _db.DeviceTokens
                    .Where(dt => invalidTokens.Contains(dt.Token))
                    .ExecuteUpdateAsync(setters => setters
                        .SetProperty(dt => dt.IsActive, false)
                        .SetProperty(dt => dt.AtualizadoEm, DateTime.UtcNow), ct);

                _logger.LogInformation("Desativados {Count} tokens invalidos", invalidTokens.Count);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao processar resposta Expo Push");
        }
    }
}

/// <summary>
/// Modelo de mensagem para Expo Push API
/// </summary>
public class ExpoPushMessage
{
    public string To { get; set; } = "";
    public string? Title { get; set; }
    public string? Body { get; set; }
    public string? Sound { get; set; }
    public Dictionary<string, object>? Data { get; set; }
    public string? Priority { get; set; }
    public string? ChannelId { get; set; }
    public int? Badge { get; set; }
    public int? Ttl { get; set; }
}
```

### 3.7 DeviceTokenController

```csharp
// Controller/DeviceTokenController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MeuApp.ExternalData;
using MeuApp.Models;

namespace MeuApp.Controller;

[ApiController]
[Route("api/device-tokens")]
public class DeviceTokenController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly ILogger<DeviceTokenController> _logger;

    public DeviceTokenController(AppDbContext db, ILogger<DeviceTokenController> logger)
    {
        _db = db;
        _logger = logger;
    }

    public class RegisterTokenRequest
    {
        public int ClienteId { get; set; }
        public string Token { get; set; } = "";
        public string Platform { get; set; } = "";
    }

    [HttpPost("register")]
    public async Task<IActionResult> RegisterToken([FromBody] RegisterTokenRequest request)
    {
        if (string.IsNullOrWhiteSpace(request.Token))
            return BadRequest("Token e obrigatorio");

        if (request.Platform != "ios" && request.Platform != "android")
            return BadRequest("Platform deve ser 'ios' ou 'android'");

        try
        {
            // Verifica se token ja existe
            var existing = await _db.DeviceTokens
                .FirstOrDefaultAsync(dt => dt.ClienteId == request.ClienteId &&
                                           dt.Token == request.Token);

            if (existing != null)
            {
                // Reativa se estava inativo
                if (!existing.IsActive)
                {
                    existing.IsActive = true;
                    existing.AtualizadoEm = DateTime.UtcNow;
                    await _db.SaveChangesAsync();
                }
                return Ok(new { message = "Token ja registrado" });
            }

            // Cria novo registro
            var deviceToken = new DeviceToken
            {
                ClienteId = request.ClienteId,
                Token = request.Token,
                Platform = request.Platform,
                IsActive = true,
                CriadoEm = DateTime.UtcNow
            };

            _db.DeviceTokens.Add(deviceToken);
            await _db.SaveChangesAsync();

            _logger.LogInformation("Token registrado para cliente {ClienteId}", request.ClienteId);
            return Ok(new { message = "Token registrado com sucesso" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao registrar token");
            return StatusCode(500, "Erro ao registrar token");
        }
    }

    [HttpPost("unregister")]
    public async Task<IActionResult> UnregisterToken([FromBody] RegisterTokenRequest request)
    {
        try
        {
            await _db.DeviceTokens
                .Where(dt => dt.ClienteId == request.ClienteId && dt.Token == request.Token)
                .ExecuteUpdateAsync(setters => setters
                    .SetProperty(dt => dt.IsActive, false)
                    .SetProperty(dt => dt.AtualizadoEm, DateTime.UtcNow));

            return Ok(new { message = "Token desregistrado" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao desregistrar token");
            return StatusCode(500, "Erro ao desregistrar token");
        }
    }
}
```

### 3.8 NotificacoesController

```csharp
// Controller/NotificacoesController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MeuApp.ExternalData;

namespace MeuApp.Controller;

[ApiController]
[Route("api/notificacoes")]
public class NotificacoesController : ControllerBase
{
    private readonly AppDbContext _db;

    public NotificacoesController(AppDbContext db)
    {
        _db = db;
    }

    [HttpGet("cliente/{clienteId}")]
    public async Task<IActionResult> ListarNotificacoes(int clienteId,
        [FromQuery] int pagina = 1, [FromQuery] int tamanhoPagina = 20)
    {
        var notificacoes = await _db.Notificacoes
            .Where(n => n.ClienteId == clienteId)
            .OrderByDescending(n => n.CriadoEm)
            .Skip((pagina - 1) * tamanhoPagina)
            .Take(tamanhoPagina)
            .Select(n => new
            {
                n.Id,
                n.Tipo,
                n.Titulo,
                n.Mensagem,
                n.Lida,
                n.CriadoEm,
                n.FeiraId,
                n.InscricaoId,
                n.DadosExtras
            })
            .ToListAsync();

        return Ok(notificacoes);
    }

    [HttpGet("cliente/{clienteId}/nao-lidas/count")]
    public async Task<IActionResult> ContarNaoLidas(int clienteId)
    {
        var count = await _db.Notificacoes
            .CountAsync(n => n.ClienteId == clienteId && !n.Lida);

        return Ok(count);
    }

    [HttpPut("{id}/marcar-lida")]
    public async Task<IActionResult> MarcarComoLida(int id)
    {
        await _db.Notificacoes
            .Where(n => n.Id == id)
            .ExecuteUpdateAsync(setters => setters.SetProperty(n => n.Lida, true));

        return Ok();
    }

    [HttpPut("cliente/{clienteId}/marcar-todas-lidas")]
    public async Task<IActionResult> MarcarTodasComoLidas(int clienteId)
    {
        await _db.Notificacoes
            .Where(n => n.ClienteId == clienteId && !n.Lida)
            .ExecuteUpdateAsync(setters => setters.SetProperty(n => n.Lida, true));

        return Ok();
    }
}
```

### 3.9 Registrar Servicos no Program.cs

```csharp
// Program.cs
builder.Services.AddHttpClient("ExpoPush", client =>
{
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.DefaultRequestHeaders.Add("Accept-Encoding", "gzip, deflate");
});

builder.Services.AddScoped<IPushNotificationService, PushNotificationService>();
```

---

## PARTE 4: Enviando Notificacoes

### 4.1 Exemplo: Notificar Nova Inscricao

```csharp
// Em InscricoesController.cs
[HttpPost]
public async Task<IActionResult> CriarInscricao([FromBody] InscricaoRequest request)
{
    // ... cria inscricao ...

    // Notifica o organizador
    var feira = await _db.Feiras.FindAsync(request.FeiraId);
    if (feira != null)
    {
        await _pushService.CreateAndSendNotificationAsync(
            clienteId: feira.OrganizadorId,
            tipo: TipoNotificacao.InscricaoRecebida,
            titulo: "Nova inscricao!",
            mensagem: $"{vendedor.Nome} se inscreveu em '{feira.Nome}'",
            feiraId: feira.Id,
            inscricaoId: inscricao.Id
        );
    }

    return Ok(inscricao);
}
```

### 4.2 Exemplo: Push sem Salvar no Historico (Chat)

```csharp
// Em ChatHub.cs - envia push mas nao salva na tabela notificacoes
await _pushService.SendPushNotificationAsync(
    clienteId: destinatarioId,
    titulo: "Nova mensagem",
    mensagem: $"{remetente.Nome}: {preview}",
    data: new Dictionary<string, object>
    {
        { "tipo", TipoNotificacao.NovaMensagem },
        { "conversa_id", conversaId }
    }
);
```

---

## PARTE 5: Testando

### 5.1 Build de Desenvolvimento

Push notifications NAO funcionam no Expo Go (Android). Crie um development build:

```bash
# Instalar EAS CLI
npm install -g eas-cli

# Login
eas login

# Criar build de desenvolvimento
eas build --profile development --platform android
```

### 5.2 Testar Endpoint Diretamente

```bash
curl -X POST https://exp.host/--/api/v2/push/send \
  -H "Content-Type: application/json" \
  -d '[{
    "to": "ExponentPushToken[XXXXX]",
    "title": "Teste",
    "body": "Mensagem de teste",
    "data": {"tipo": "teste"}
  }]'
```

### 5.3 Logs para Debug

No backend, adicione logs detalhados:

```csharp
_logger.LogInformation("PUSH: Enviando para {ClienteId}, Titulo='{Titulo}'",
    clienteId, titulo);
_logger.LogInformation("PUSH: Tokens encontrados: {Count}", tokens.Count);
_logger.LogInformation("PUSH: Response: {Status} - {Body}",
    response.StatusCode, responseBody);
```

---

## PARTE 6: Checklist Final

### Firebase
- [ ] Projeto criado no Firebase Console
- [ ] App Android adicionado com package correto
- [ ] App iOS adicionado com bundle ID correto (se aplicavel)
- [ ] google-services.json na raiz do projeto Expo
- [ ] GoogleService-Info.plist na raiz (iOS)
- [ ] Service Account Key gerada (FCM V1)
- [ ] Service Account Key configurada no expo.dev

### Expo/React Native
- [ ] expo-notifications instalado
- [ ] expo-device instalado
- [ ] expo-constants instalado
- [ ] app.json configurado com plugins e googleServicesFile
- [ ] projectId do Expo configurado em extra.eas
- [ ] notificationService.ts implementado
- [ ] NotificationContext.tsx implementado
- [ ] NotificationProvider adicionado no App

### Backend .NET
- [ ] Modelo DeviceToken criado
- [ ] Modelo Notificacao criado (opcional)
- [ ] TipoNotificacao constantes definidas
- [ ] DbContext atualizado com DbSets
- [ ] Migration executada no banco
- [ ] PushNotificationService implementado
- [ ] DeviceTokenController implementado
- [ ] NotificacoesController implementado
- [ ] HttpClient "ExpoPush" registrado
- [ ] IPushNotificationService registrado no DI

### Testes
- [ ] Build de desenvolvimento criado (nao Expo Go)
- [ ] Token sendo registrado no backend
- [ ] Push chegando no dispositivo
- [ ] Navegacao funcionando ao tocar na notificacao
- [ ] Tokens invalidos sendo desativados

---

## Erros Comuns

### "Unable to retrieve the FCM server key"
**Causa**: FCM nao configurado no Expo ou usando Server Key legada
**Solucao**: Configure FCM V1 Service Account Key no expo.dev

### Push nao chega no Android
**Causa**: Testando no Expo Go
**Solucao**: Crie um development build com EAS

### "DeviceNotRegistered"
**Causa**: Token expirado ou app desinstalado
**Solucao**: O servico ja desativa automaticamente esses tokens

### Navegacao falha ao tocar na notificacao
**Causa**: Dados incompletos para a tela de destino
**Solucao**: Busque dados necessarios antes de navegar (ex: dados da conversa)

### Badge nao atualiza
**Causa**: Permissao de badge nao concedida
**Solucao**: Verifique permissoes no iOS (Settings > Notifications > App)

---

## Referencias

- [Expo Push Notifications](https://docs.expo.dev/push-notifications/overview/)
- [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)
- [Expo Push API](https://docs.expo.dev/push-notifications/sending-notifications/)
- [EAS Build](https://docs.expo.dev/build/introduction/)
