# Websockety

WebSocket je komunikačný protokol, ktorý umožňuje vytvoriť trvalé obojsmerné spojenie medzi klientom (napríklad webovým prehliadačom) a serverom. Na rozdiel od klasického HTTP protokolu, kde klient vždy iniciuje požiadavku a server odpovedá, pri WebSockete môže klient aj server posielať správy kedykoľvek bez potreby opakovaného nadväzovania spojenia.

Samotné spojenie sa začína ako bežná HTTP požiadavka, ktorá obsahuje hlavičku `Upgrade: websocket`. Server, ak podporuje WebSockety, odpovie stavovým kódom `101 Switching Protocols`, čím dôjde k zmene spojenia na WebSocket. Od tej chvíle už komunikácia prebieha ako nepretržitý, obojsmerný dátový tok.

WebSockety sa používajú najmä v aplikáciách, kde je potrebná okamžitá reakcia alebo neprerušovaná obojsmerná komunikácia, napríklad pri chatoch, online hrách alebo live notifikáciách.

## Trocha potrebnej teórie

Náš WebSocket server bude postavený na Node.js. Ide o runtime prostredie, ktoré umožňuje spúšťať JavaScript mimo webového prehliadača, priamo na serveri. Je postavené na V8 engine (rovnaký engine, ktorý používa prehliadač Google Chrome). Node.js používa `npm` ako správcu balíkov (podobne ako Composer pre PHP). Analogicky je možné implementovať WebSocket server aj prostredníctvom iných runtime prostredí, napr. Deno, Bun (TypeScript), Python, prípadne v jazyku PHP použiť Laravel/Reverb, Workerman, Ratchet, Swoole.

**Pozor**: Pri WebSocket komunikácii je nevyhnutné uvedomiť si zásadnú vec: WebSocket **nie je štadnardne interpretovaný skript pre webový server (Nginx/Apache).** Ide o samostatnú službu, ktorá beží nezávisle na webovom serveri. V operačnom systéme teda musí bežať ako samostatný proces (daemon). To znamená, že sa spúšťa cez `CLI` a pri dlhodobej prevádzke je jeho beh riadený systémovým správcom procesov napr. `supervisor` alebo `systemd`. 

> Prečo používame Node.js? V kontexte WebSocketov je situácia v PHP trochu špecifická, pretože PHP bolo pôvodne navrhnuté pre klasický request–response model (HTTP), nie pre dlhodobé otvorené spojenia. To znamená, že na rozdiel od Node.js alebo iných JS/TS/Python runtime prostredí, kde sú WebSockety prirodzenou súčasťou ekosystému, v PHP je potrebné použiť dodatočné knižnice alebo špeciálne nástroje.

## Inštalácia Node.js

Na začiatok si aktualizujte systémové repozitáre a systém (ak tak nerobíte pravidelne):
```sh
sudo apt update && sudo apt -y upgrade
```

> V pripade inštalácie dodatočných balíkov na server alebo zmene konfigurácie serveru alebo systému nezabudnite uviesť konkrétne zmeny v dokumentácii k zadaniu.

Node.js môžeme jednoducho nainštalovať pomocou [oficiálneho návodu](https://nodejs.org/en/download). V našom prípade budeme inštalovať `v24.14.1 (LTS)` pre Linux pomocou `nvm` - _Node Version Manager_ spolu so správcom balíkov `npm` - _Node Package Manager_. Nasledovné príkazy spustíme na serveri po jednom, nepotrebujeme `sudo`:

Stiahnutie a inštalácia `nvm`:

```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

Aby sme sa nemuseli odhlasovať a prihlasovať, zavoláme príkaz:

```sh
\. "$HOME/.nvm/nvm.sh"
```

Stiahneme a nainštalujeme Node.js:

```sh
nvm install 24
```

Po inštalácii môžeme overiť, či je všetko tak, ako očakávame:

```sh
node -v # Mal by vypisat "v24.14.1" alebo podobne
npm -v # Mal by vypisat "11.11.0" alebo podobne
```

## Jednoduchý príklad

V tomto príklade si vytvoríme jednoduchú chatovaciu aplikáciu. Klient (prehliadač) sa bude pripájať cez zabezpečený protokol `wss://` na server, pričom komunikácia bude prechádzať cez Nginx, ktorý nakonfigurujeme ako reverzný proxy server a zároveň bude zabezpečovať SSL certifikát. Samotná logika WebSocket serveru bude implementovaná v `Node.js` aplikácii bežiacej na porte 3000.

### WebSocket Server

Adresár s WebSocket serverom vytvoríme _mimo_ koreňový adresár serveru, v našom prípade v `/home/xusername/ws-chat`:

```sh
cd && mkdir ws-chat
```

> Adresár s Node aplikáciou (`ws-chat` v našom prípade) vieme vytvoriť aj vo `/var/www`, ale nie je to vždy najlepšia voľba. Záleží na tom, ako chceme mať server organizovaný. Adresár `/var/www` sa štandardne používa pre statické súbory (HTML, CSS, JS) a PHP aplikácie obsluhované cez Nginx/Apache. Je to "web root", ktorý Nginx priamo obsluhuje. Naša aplikácia (1) nebeží cez Nginx, (2) beží ako samostatný proces - daemon a (3) Nginx na ňu cez proxy odošle požiadavku.

Presunieme sa do vytvoreného adresára, inicializujeme si `npm` prostredie a nainštalujeme knižnicu pre prácu s WebSocketmi.

```sh
cd ws-chat && npm init -y && npm install ws
```

> Nezabudnite pri odovzdávaní pribaliť aj adresár s Node aplikáciou - WebSocket server. Samozrejme **bez adresára node_modules**. Node aplikácie sa distribuujú iba s `package.json` a `package-lock.json`.

Následne si vytvoríme súbor `server.js`, v ktorom implementujeme logiku WebSocket serveru:

```js
// Nacitanie kniznice ws, ktora implementuje WebSocket server
const WebSocket = require('ws');

// Vytvorenie WebSocket servera na porte 3000
const wss = new WebSocket.Server({ port: 3000 });

// Pole na uchovavanie vsetkych pripojenych klientov
let clients = [];

// Udalost, ktora sa vykona pri novom pripojeni klienta
wss.on('connection', (ws) => {
    console.log('New client connected');

    // Pridanie klienta do zoznamu
    clients.push(ws);

    // Udalost, ktora sa vykona pri prijati spravy od klienta
    ws.on('message', (message) => {
        const msg = message.toString();
        console.log('Received message:', msg);

        // Rozposlanie spravy vsetkym pripojenym klientom (broadcast)
        clients.forEach(client => {
            // Kontrola, ci je spojenie stale aktivne
            if (client.readyState === WebSocket.OPEN) {
                client.send(msg);
            }
        });
    });

    // Udalost, ktora sa vykona pri odpojeni klienta
    ws.on('close', () => {
        console.log('Client disconnected');

        // Odstranenie klienta zo zoznamu
        clients = clients.filter(c => c !== ws);
    });
});

// Informacia do konzoly, ze server bezi
console.log("+------------------------------------------+");
console.log("|  WebSocket server running on port 3000   |");
console.log("+------------------------------------------+");
```

Server spustíme príkazom v konzole:

```sh
node server.js
```

### Klient

Na strane klienta vytvoríme jednoduchý HTML súbor, ktorý sa pripojí na WebSocket server a umožní odosielať a prijímať správy. Nemusíme nutne inštalovať žiadnu knižnicu na strane klienta, pretože WebSocket je natívne podporovaný priamo v moderných webových prehliadačoch. WebSocket API je súčasťou štandardu webových technológií (rovnako ako napr. `fetch` alebo `DOM`).

Vytvoríme si súbor v koreňovom adresári webového serveru, napríklad v adresári `chat-app`:

```html
<!DOCTYPE html>
<html lang="sk">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebSocket Chat</title>
</head>
<body>

<h2>WebSocket Chat</h2>

<!-- Vstupne pole na spravu -->
<input id="msg" placeholder="Napíš správu">

<!-- Tlacidlo na odoslanie -->
<button onclick="sendMsg()">Odoslať</button>

<!-- Zoznam sprav -->
<ul id="chat"></ul>

<script>
    // Vytvorenie WebSocket spojenia cez zabezpecený protokol wss
    const ws = new WebSocket("wss://nodeXX.webte.fei.stuba.sk/ws/");

    // V nasom pripade, kedy klientsky kod hostujeme z toho isteho systemu, mozeme sa
    // na WS pripojit aj tym, ze connection string bude iba "/ws":
    // const ws = new WebSocket("/ws");

    // Udalost pri prijati spravy zo servera
    ws.onmessage = (event) => {
        const li = document.createElement("li");
        li.textContent = event.data;
        document.getElementById("chat").appendChild(li);
    };

    // Funkcia na odoslanie spravy na server
    function sendMsg() {
        const input = document.getElementById("msg");
        ws.send(input.value);
        input.value = "";
    }
</script>

</body>
</html>
```

V konfiguračnom súbore pre Nginx je potrebné nastaviť reverzný proxy pre WebSocket spojenia. Do server bloku pridáme lokalitu `/ws/`, ktorá bude presmerovávať požiadavky na Node.js server. 

Prekonfigurovanie Nginxu na reverse proxy je pri WebSocketoch potrebné preto, že samotný Nginx nevie vykonávať logiku WebSocket servera. V tomto prípade funguje ako sprostredkovateľ medzi klientom a backend aplikáciou.

Keď používateľ otvorí stránku: https://nodeXX.webte.fei.stuba.sk/chat-app, prehliadač komunikuje s Nginx serverom (HTTP). Avšak, naša WebSocket aplikácia (server) beží napríklad na: `localhost:3000`. Klient sa k nemu nevie pripojiť priamo (a ani by nemal).

Čo robí reverse proxy? Reverse proxy znamená, že Nginx (1) prijme požiadavku od klienta, následne ju (2) presmeruje na backend (Node.js WS server) a (3) pošle odpoveď späť klientovi. WebSocket nie je obyčajný HTTP request. Používa tzv. upgrade mechanizmus. Ak nebude Nginx korektne nakonkfigurovaný: 
1. neprebehne upgrade spojenia `HTTP → WebSocket`,
2. WebSocket sa neotvorí,
3. dostaneme chybu HTTP 400 alebo 502.

> Teoreticky je pri lokálnom vývoji možné použiť aj priame pripojenie na WebSocket, tzn. v klientskom JavaScript kóde použiť priamo `new WebSocket("ws://localhost:3000")`. V prípade nasadenej aplikácie na produkčnom serveri nie je tento postup odporúčaný, pretože: port 3000 nemusí byť otvorený, nemáme natívne SSL (`wss://`), ide o bezpečnostné riziko ak by sme otvárali port vynútene, môžu byť problémy s firewallom. Navyše, keďže používame HTTPS, prehliadač bude vyžadovať zabezpečené WebSocket pripojenie. SSL certifikát má Nginx (aj keď je možné ho namapovať aj priamo WS serveru). 

Do existujúcej konfigurácie Nginx serveru pridáme (niekde za ostatné `location` bloky):

```conf
location /ws/ {
    # Presmerovanie na Node.js server
    proxy_pass http://localhost:3000/;

    # Povolenie WebSocket upgrade mechanizmu
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";

    # Zachovanie hlavicky hosta.
    proxy_set_header Host $host;
}
```

> V pripade inštalácie dodatočných balíkov na server alebo zmene konfigurácie serveru alebo systému nezabudnite uviesť konkrétne zmeny v dokumentácii k zadaniu.

Po úprave konfigurácie je potrebné Nginx reštartovať:

```sh
sudo nginx -t
sudo systemctl restart nginx
```

## Automatické spustenie WS serveru

Aby sa WebSocket aplikácia (server) automaticky spustila napr. aj po reštarte systému, vytvoríme `systemd` službu. Tento prístup je stabilnejší a vhodnejší pre produkčné prostredie než spúšťanie cez terminál.

Najskôr vytvoríme nový service súbor:

```sh
sudo nano /etc/systemd/system/ws-chat.service
```

Do súboru vložíme nasledovnú konfiguráciu:

```ini
[Unit]
Description=WebSocket Chat Server
After=network.target

[Service]
# Priečinok, kde sa nachádza aplikácia
WorkingDirectory=/home/USERNAME/ws-chat

# Príkaz na spustenie servera
ExecStart=NODEJS_PATH server.js

# Automatické reštartovanie pri páde
Restart=always

# Používateľ, pod ktorým sa služba spúšťa
User=USERNAME

[Install]
WantedBy=multi-user.target
```

**Pozor:** Všimnime si `USERNAME` a `NODEJS_PATH` - namiesto týchto placeholderov musíme doplniť korektné údaje. `USERNAME` je potrebné nahradiť vašim loginom, teda `xmrkvickaj` na dvoch miestach:

1. v ceste k adresáru s WebSocket aplikáciou: `WorkingDirectory=/home/xmrkvicka/ws-chat`
2. v používateľovi, pod ktorým sa služba spustí: `User=xmrkvicka`

Následne je potrebné nahradiť `NODEJS_PATH` cestou, na ktorej sa nachádza nainštalovaný Node.js runtime. Túto cestu vieme zistiť príkazom:

```sh
which node
```

ktorý vráti absolútnu cestu k Node.js - táto cesta môže byť iná v prípade vašej inštalácie (v závislosti od toho, čo ste inštalovali a akým spôsobom):

```
/home/xmrkvicka/.nvm/versions/node/v24.14.1/bin/node
```

Túto cestu použíjeme pre konfiguráciu príkazu na spustenie serveru v konfigurácii systémovej služby, teda: `ExecStart=/home/xmrkvicka/.nvm/versions/node/v24.14.1/bin/node server.js`

Po uložení konfigurácie musíme novú službu načítať:

```sh
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
```

Službu môžeme spustiť príkazom:

```sh
sudo systemctl start ws-chat
```

alebo zastaviť príkazom:

```sh
sudo systemctl stop ws-chat
```

Ak chceme, aby sa služba spustila automaticky po štarte systému (týmto spôsobom zabezpečíme, že WebSocket server bude bežať nepretržite a automaticky sa spustí po každom reštarte VPS):

```sh
sudo systemctl enable ws-chat
```

prípadne zakázanie automatického spustenia: 

```sh
sudo systemctl disable ws-chat
```

> V pripade inštalácie dodatočných balíkov na server alebo zmene konfigurácie serveru alebo systému nezabudnite uviesť konkrétne zmeny v dokumentácii k zadaniu.

Stav služby aj výpis jej logov (alebo toho, čo vypisujeme cez `console.log`) môžeme overiť:

```sh
sudo systemctl status ws-chat
● ws-chat.service - WebSocket Chat Server
     Loaded: loaded (/etc/systemd/system/ws-chat.service; disabled; preset: enabled)
     Active: active (running) since Thu 2026-04-09 21:56:54 CEST; 1s ago
   Main PID: 1410 (MainThread)
      Tasks: 7 (limit: 4653)
     Memory: 55.9M (peak: 56.2M)
        CPU: 252ms
     CGroup: /system.slice/ws-chat.service
             └─1410 /home/xmrkvicka/.nvm/versions/node/v24.14.1/bin/node server.js

Apr 09 21:56:54 node10 systemd[1]: Started ws-chat.service - WebSocket Chat Server.
Apr 09 21:56:55 node10 node[1410]: +------------------------------------------+
Apr 09 21:56:55 node10 node[1410]: |  WebSocket server running on port 3000   |
Apr 09 21:56:55 node10 node[1410]: +------------------------------------------+
```
