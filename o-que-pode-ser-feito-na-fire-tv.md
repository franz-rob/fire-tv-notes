# O que pode ser feito numa Fire TV sem root

English: [README.md](README.md)

Testado em novembro de 2026 num aparelho, com ADB (`shell`, uid 2000) e um app comum. Sem root.

| | |
|---|---|
| Aparelho | Fire TV Stick Lite (2020), AFTSS "sheldon" |
| Sistema | Fire OS 7.7.1.7 (PS7717/5746), Android 9 (API 28) |
| Controle | Alexa Voice Remote Lite: botão de TV e 4 botões de apps, **sem botões de volume e de energia** |
| TV | Chip MStar, HDMI-CEC 1.4, sem soundbar |

Outros aparelhos com Fire OS 7 podem se comportar igual, mas isso não foi testado. O Fire OS 8 (Android 11) restringe abrir telas em segundo plano, então parte disto pode não valer nele.

Toda alteração abaixo tem como desfazer, e o comando está junto. O último recurso é o reset de fábrica: segure **Voltar + seta direita por 10 s** no controle.

## Resumo

| Objetivo | Resultado | Como |
|---|---|---|
| Trocar o launcher da Amazon | ❌ | Pacote protegido, e o `set-home-activity` é ignorado (seção 1) |
| Abrir o seu app ao ligar, antes da home da Amazon | ✅ visível aos ~22 s | Serviço de acessibilidade (seção 2) |
| Trazer o app de volta quando a home da Amazon aparece | ✅ ~5 s depois do Home | Serviço de acessibilidade, eventos de janela |
| Bloquear Home, Alexa, TV (guia) e os botões de apps | ✅ | `RESTRICT_REMOTE_BUTTONS=1` (seção 3) |
| Manter o bloqueio depois de reiniciar | ✅ | O app religa (`WRITE_SECURE_SETTINGS`) |
| Dar outra função a Home, Alexa, TV ou botões de apps | ❌ | O sistema trata essas teclas antes de qualquer app |
| Usar Menu, ⏪, ⏯, ⏩ em qualquer tela | ✅ | Filtro de teclas da acessibilidade |
| Usar as teclas do controle da TV (pelo CEC) | ✅ só no seu app | `dispatchKeyEvent` |
| Ligar a TV junto com o Stick | ✅ só pelo controle do Fire TV | Acordar o Stick liga a TV |
| Desligar a TV quando o Stick dorme | ❌ | O Fire OS não manda `<Standby>` |
| Fazer o Stick dormir quando a TV desliga | ✅ | A TV manda `<Standby>` |
| Mudar o volume da TV por um app | ❌ | Seção 5 |
| Desativar apps da Amazon em segundo plano | ✅ vários | `pm disable-user` (seção 6) |
| Impedir atualizações do Fire OS | ⚠️ em parte | Seção 7 |
| Remover a animação de boot | ❌ | Fica em `/system`, só leitura |

## 1. O launcher da Amazon não pode ser trocado

- `pm disable-user --user 0 com.amazon.tv.launcher` → `SecurityException: Cannot disable a protected package`.
- `cmd package set-home-activity <seu.app>/.HomeActivity` responde `Success` e não muda nada.
- Candidatos a HOME, por prioridade: `com.amazon.tv.launcher/.ui.HomeActivity_vNext` (950), `com.amazon.firehomestarter/.HomeStarterActivity` (1), um app de terceiros (0), `com.amazon.tv.settings.v2/.system.FallbackHome` (-1000). Declarar `HOME` + `DEFAULT` não ajuda.
- Existe um gancho oficial de home própria: o `com.amazon.policy.keypolicymanager.CustomHomeButtonHandler` procura no app da frente o meta-data `com.amazon.metadata.CUSTOM_HOME` e manda `com.amazon.tv.action.CUSTOM_HOME`. Ele exige `com.amazon.permission.RECEIVE_CUSTOM_HOME`, que é `signature|amazon`. A cada Home, o log mostra `CustomHomeButtonHandler: getCustomHomeReceiver: application info meta data is null`. (Lido no `/system/framework/oat/arm/fosservices.vdex`; não testado com um app.)

## 2. Abrir o seu app ao ligar

**Use um serviço de acessibilidade.** O Android liga esses serviços cedo no boot, porque leitores de tela precisam funcionar desde a primeira tela. No Android 9, um serviço assim pode abrir telas.

| Momento (uptime) | O que acontece |
|---|---|
| ~21,2 s | O serviço de acessibilidade conecta (`onServiceConnected`) e abre a sua activity |
| ~22,6 s | A activity aparece, logo depois da animação de boot |
| ~28,7 s | O launcher da Amazon sobe por trás e não vem para a frente |
| ~52–65 s | A Amazon volta o `RESTRICT_REMOTE_BUTTONS` para `0` (seção 3) |
| ~60–68 s | O `BOOT_COMPLETED` chega aos apps de terceiros (é nele que os apps de "abrir no boot" se apoiam) |

Ligar o serviço pelo ADB:
```
adb shell settings put secure enabled_accessibility_services <pacote>/<pacote>.<Servico>
adb shell settings put secure accessibility_enabled 1
```
Desfazer: `adb shell settings delete secure enabled_accessibility_services` e `adb shell settings put secure accessibility_enabled 0`. Se já houver outros serviços na lista, mantenha-os, separados por `:`.

O que o serviço deve fazer:
- Em `onServiceConnected()`, abrir a activity e tentar de novo a cada 5 s até ela estar na frente.
- Num `TYPE_WINDOW_STATE_CHANGED` do `com.amazon.tv.launcher`, abrir de novo (no máximo uma vez por segundo; nada se o app já está na frente).
- Para saber qual pacote está na frente fora de um evento, declarar `android:canRetrieveWindowContent="true"`.

Armadilhas:
- **~5 s depois do Home.** Depois de um Home, o Android adia em ~5 s a abertura de telas de outros apps (proteção de troca de app), inclusive pelo serviço de acessibilidade. Bloquear o Home (seção 3) evita isso.
- **`am force-stop <seu.app>` desliga o serviço de acessibilidade.** Religue com os dois comandos acima.
- **Não faça a abertura no boot depender de um ajuste.** Com uma opção de "redirecionar o Home" desligada, o app só subia aos ~60 s, pelo `BOOT_COMPLETED`.
- **A rede não está pronta aos ~22 s**: o DNS falha (`UnknownHostException`). Espere a rede ou tente de novo.
- **O relógio volta atrasado a cada boot** (sem bateria; acerta depois pela rede). Para temporizadores, use `SystemClock.elapsedRealtime()` e, para distinguir um boot do outro, `Settings.Global.BOOT_COUNT`.
- **Os logs somem logo depois do boot** (o `SPAM_DETECTOR` do logd). Grave um arquivo de log em `getExternalFilesDir(null)` (`/sdcard/Android/data/<pacote>/files/`) e leia com `adb pull`. Não precisa de `debuggable`.
- **App fechado morre rápido.** Depois do `finish()`, o Fire OS mata o processo em cache ~1 s depois, antes do `onStop`. Salve antes.
- **Acordar da espera.** Ao acordar, o launcher da Amazon pode voltar para a frente. O retorno pela mudança de janela, descrito acima, traz o app de volta (visto depois de esperas de até ~10 min). Por segurança, o app também confere a janela da frente 2 s e 8 s depois de acordar. A única vez em que o app não voltou, numa noite, o serviço não estava rodando.

**As Configurações da Fire TV ficam dentro do launcher.** `Settings.ACTION_SETTINGS` abre `com.amazon.tv.launcher/.ui.SettingsActivity`, que mostra as Configurações dentro da `HomeActivity_vNext`. Um app que volta sozinho sempre que o launcher aparece precisa pausar isso enquanto a pessoa está nas Configurações. Uma pausa de 5 min, contada com `elapsedRealtime` e cancelada ao reiniciar, funcionou: no fim, o app voltava se o launcher estivesse na frente e, se não, esperava o launcher aparecer de novo.

## 3. Bloquear Home, Alexa, TV e os botões de apps

Uma configuração que a Amazon usa na configuração inicial (OOBE):
```
adb shell settings put global com.amazon.tv.oobe.RESTRICT_REMOTE_BUTTONS 1
```
Desfazer: o mesmo com `0`.

- Home, Alexa, TV (guia) e os 4 botões de apps param de agir. Só aparece uma barrinha de status da Amazon embaixo.
- Setas, OK, Voltar, Menu e ⏪ ⏯ ⏩ continuam funcionando.
- O log mostra `FireTVKeyPolicyManager: block remote keys setting is enabled, won't process home key`.
- Um Home mandado por `adb shell input keyevent` também é bloqueado.

**A Amazon desfaz a cada boot.** O `com.amazon.tv.oobe/.BootCompleteReceiver` volta para `0` entre ~52 e ~65 s. Para manter:
1. Declare `android.permission.WRITE_SECURE_SETTINGS`. Ela é `signature|privileged|development`, então o ADB pode concedê-la: `adb shell pm grant <pacote> android.permission.WRITE_SECURE_SETTINGS` (desfazer: `pm revoke`).
2. Registre um `ContentObserver` em `Settings.Global.getUriFor("com.amazon.tv.oobe.RESTRICT_REMOTE_BUTTONS")` e ponha `1` de novo quando virar `0`. Em todos os reinícios testados, voltou em menos de 1 s (6 ms na última medição).

Contrapartida: com o bloqueio, o Home não faz nada. Ele não leva a lugar nenhum, então o app precisa ter o próprio caminho até a home da Amazon e as Configurações.

## 4. Teclas do controle

Controle do Fire TV: `dev="Amazon Fire TV Remote"`, src `0x301`. Controle da TV pelo HDMI-CEC: `dev="Virtual"`, id `-1`, src `0x2000001`.

| Tecla | Código | Filtro da acessibilidade (`onKeyEvent`) | App na frente (`dispatchKeyEvent`) |
|---|---|---|---|
| Fire TV: setas / OK | 19–22 / 23 | ✅ | ✅ |
| Fire TV: Voltar | 4 | ✅ | ✅ |
| Fire TV: Menu | 82 | ✅ e dá para **consumir** | ✅ |
| Fire TV: ⏪ ⏯ ⏩ | 89 / 85 / 90 | ✅ e dá para **consumir** | ✅ |
| Fire TV: Home, Alexa, TV, botões de apps | — | ❌ | ❌ (o sistema trata antes) |
| Controle da TV (CEC): setas, OK, ⏪ ⏩ ⏸ ⏹ ⏮ ⏭ | 19–23, 89, 90, 127, 86, 88, 87 | ❌ | ✅ |

- Consumir (retornar `true` em `onKeyEvent`) exige `android:canRequestFilterKeyEvents="true"` e `flagRequestFilterKeyEvents`.
- Teclas de `adb shell input keyevent` não chegam ao filtro da acessibilidade, só ao app da frente. Teste o filtro com o controle de verdade.
- Os botões de apps são "partner keys": o mapa botão → app vem da nuvem da Amazon (`partnerkeyconfig.json`), e o sistema abre `com.amazon.venezia/.pdi.AppLaunchActivity`. O botão TV (guia) abre `com.amazon.tv.livetv/.GuideWrapperActivity`.

## 5. HDMI-CEC: energia e volume da TV

Um app comum não manda CEC: isso exige `HDMI_CEC`, uma permissão só de apps do sistema. O ADB também não manda.

| Funciona | Como |
|---|---|
| Ligar a TV ao acordar o Stick | Só pelo controle do Fire TV: o OK acorda o Stick e a TV liga. Quem manda é o serviço HDMI da Amazon, fora do histórico do `HdmiControlService`. Um app não acorda o Stick |
| Fazer o Stick dormir quando a TV desliga | A TV manda `<Standby>` (`0F:36`), e o Stick dorme em ~1 s |
| Teclas do controle da TV no app | A TV repassa pelo CEC |

| Não funciona | Por quê |
|---|---|
| Desligar a TV quando o Stick dorme | O Fire OS não manda `<Standby>`, nem com `hdmi_control_auto_device_off_enabled=1` (`mAutoTvOff:true`). O Stick só corta o sinal HDMI, e a TV fica ligada, "sem sinal" |
| Volume da TV por um app | `AudioManager.adjustStreamVolume`, `adjustSuggestedStreamVolume` e `VOLUME_UP/DOWN` simulados não mandam nada (`STREAM_MUSIC` fica fixo em 15/15 no HDMI). Só o "Controle de equipamentos" da Amazon (`com.amazon.tv.devicecontrol`) manda volume pelo CEC, e as Configurações o mostram como indisponível com este controle: ele precisa de um controle com botões de energia e volume, ou de um Fire TV Blaster. Forçar `dc_volume_control_enabled=1` não mudou nada |

## 6. Apps da Amazon em segundo plano

Cada um foi desativado com `adb shell pm disable-user --user 0 <pacote>`, um por vez e com reinício depois. Todos foram aceitos, e o aparelho continuou funcionando. Desfazer: `adb shell pm enable <pacote>`.

| Pacote | O que é |
|---|---|
| `com.amazon.bueller.photos` | Amazon Photos |
| `com.amazon.tahoe` | Amazon Kids |
| `com.amazon.whad` | Áudio multiambiente (PERSISTENT: continua sendo iniciado no boot) |
| `com.amazon.aca` | Alexa Central for Apps |
| `com.amazon.ceviche` | Métricas |
| `com.amazon.tv.legal.notices` | Avisos legais |
| `com.amazon.whisperjoin.middleware.np`, `com.amazon.tv.ffsprovisioneeclient` | Configuração automática de Wi-Fi |
| `com.amazon.tv.developer.dataservice` | Serviço de dados para desenvolvedores |
| `com.amazon.avsyncslider` | Ajuste de sincronia de áudio e vídeo |
| `com.amazon.tv.forcedotaupdater.v2` | Atualizador forçado do Fire OS |
| `com.amazon.vizzini` | Núcleo da Alexa (PERSISTENT: continua rodando, e **a Amazon o reativa no boot**) |
| `com.amazon.vizzini.ftvcds`, `com.amazon.alexamediaplayer.runtime.ftv`, `com.amazon.alexa.externalmediaplayer.fireos`, `com.amazon.alexadirectivebrokerservice`, `com.amazon.alexa.datastore.app`, `com.amazon.tv.alexanotifications`, `com.amazon.tv.alexaalerts`, `com.amazon.aria` | Alexa e Alexa+ |
| `com.amazon.firebat` | Prime Video |

- **Protegidos** (`Cannot disable a protected package`): `com.amazon.tv.launcher`, `com.amazon.diode`, `com.amazon.device.software.ota`, `com.amazon.device.software.ota.override`.
- **Deixados de propósito:** `com.amazon.tv.devicecontrol` (controle de equipamentos pelo CEC), `com.amazon.device.blepa` e `com.amazon.accessoryconfigservice` (pareamento do controle: você pode ficar sem controle), `com.amazon.tv.ime` (teclado), `com.amazon.logan` (leitor de tela VoiceView), `com.amazon.prism.android.service`.

## 7. Atualizações do Fire OS

| Tentativa | Resultado |
|---|---|
| `pm disable-user` no `com.amazon.device.software.ota` e no `...ota.override` | ❌ Protegidos |
| `pm disable-user` no `com.amazon.tv.forcedotaupdater.v2` | ✅ Desativado |
| `pm suspend com.amazon.device.software.ota` | ❌ O `shell` não tem `SUSPEND_APPS` |
| `cmd appops set com.amazon.device.software.ota RUN_IN_BACKGROUND ignore` | ⚠️ Aceito e mantido depois do reinício. Barrou ao menos uma execução em segundo plano, mas o `OtaService` sobe no boot |

Ainda não testado: bloquear os endereços de atualização no roteador ou com DNS privado.

## Ainda em aberto

- Confirmar com o controle de verdade que Home e os botões de apps continuam bloqueados depois do reinício.
- Desligar a TV pelo Stick, e acordar o Stick por um app.
- Dar outra função a Home, Alexa, TV e botões de apps (por enquanto só dá para bloqueá-los).
