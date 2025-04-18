# Opis projektu
<p>Notatka z lekcji poprawiona w .md + uściślenie niekótrych kwestii. <br>
Można traktować jak dokumentacje</p>

## Ogólne założenia
Projekt obejmuje aplikację do wysyłania wiadomości, zakłada server i klient, w komunikacji jeden do wielu. Protokół komunikacji sieciowej to TCP. Generalne szyfrowanie wiadomości odbywa się za pomocą algorytmów klucza publicznego i AES (dla zapewnienia większej długosći wiadomości). Więcej informacji o tym w sekcji [komunikacji z serwerem](#przesyłanie-wiadomości) i sekcji o [strukturze wiadomości](#message).  

### Server 
Server odpowiada tylko i wyłącznie za przekazywanie wiadomości między użytkownikami i nie ma wglądu do treści wiadomości. 
Wiadomości są przechowywane w bazie danych (w formie zaszyfrowanej). Server posiada takrzę baze danych z listą użytkowników oraz ich kluczami publicznymi. Klucze prywatne pozostają prywatne.
Server podczas działania przechowuje także w cashu listę aktywnie zalogowanych użytkowników. Lista ta zawiera dokładniej nazwę użytkownika, token sesji i socket.

### Klient
Wiadomości między użytkownikami są szyfrowane. Dzięki wykorzystaniu algorytmów klucza publicznego, użytkownicy do logowania wymagają tylko swojej nazwy i pary kluczy które są przechowywane na urządzeniu. 
W obecnym momencie nie jest przewidziane automatyczne urzytkowanie kluczy na wielu urządzeniach równocześnie.

## Opis teoretyczny komunikacji klienta z serverem
Wszystkie zapytania do servera będą przesyłane za pomcą Jsonów.

### Rejestracja
Proces utworzenia nowego użytkownika i dodania go do bazy danych użytkowników na serverze.

#### Użytkownik
User generuje sobie parę RSA (klucz publiczny i prywatny) wysyła na serwer publiczne RSA + nazwa użytkownika i serwer to zapisuje. Następnie wysyła zapytanie do Servera ze swoim username + kluczem i oczekuje odpowiedzi.
#### Server
Serwer sprawdza czy nazwa użytkownika jest zajęta, jeśli jest zajęta to wysyła do użytkownika, że rejestracja się nie powiodła, jeżeli nie jest zajęta wysyła info zwrotne że się powiodła. 

**Użytkownik po rejestracji nie jest zalogowany.**

### Logowanie
Proces uwierzytelnienia sesji użytkownika i zmiana jego stanu na online.
#### Użytkownik
Użytkownik wysyła do serwera request, z prośbą o logowanie (+ swój username). Oczekuje na odpowiedź serwera. Dostaje token zaszyfrowany swoim Publicznym RSA. Rozszyfrowywuje go i szyfruje ponownie, swoim kluczem prywatym i odsyła do serwera. Czeka na autoryzacje.

#### Server
Server odbiera request logowania. Generuje token (losowe znaki) i szyfruje kluczem RSA publicznym użytkownika, i wysyła do użytkownika. Oczekuje odpowiedzi. Po dostaniu odpowiedzi, rozszyfrowywuje ją za pomocą klucza publicznego RSA użytkownika. Sprawdza czy tokeny się zgadzają. Odsyła informację zwrotną do użytkownika, czy logowanie się powiodło.

### Wylogowanie
Proces zmiany stanu użytkownika na offline i anulowanie połączenia.
#### Użytkownik
Użytkownik wysyła request wylogowania się.
#### Server
Server odbiera request wylogowania się od użytkownika, zmienia jego stan na offline.

### Przesyłanie wiadomości
Treść wiadomości wysłanej przez użytkownika, będzie zaszyfrowana algorytmem AES, który będzie zaszyfrowany kluczem publicznym nadawcy i odbiorcy/ów, żeby pozwolić na większą długość wiadomość.

**UWAGA: przy obecnej implementacji nie ma gwarancji tego że osoba wysyłająca zapytanie do servera jest tą osobą za którą się podaje**  
**jednak przy poprawnej implementacji klienta, próba podszycia się zakończy się co najwyżej wiadomością składającą się ze śmieci.**

#### Użytkownik
Klucz AES jest generowany przez klienta. Następnie jest nim szyfrowana treść wiadomości. Później klucz AES jest szyfrowany za pomocą klucza publicznego odbiorcy (lub odbiorców). Cała wiadomość jest spakowana w odpowiednią strukturę JSON a następnie przesłana do servera.

#### Server
Server po odebraniu wiadomości, zapisuje ją w lokalnej bazie danych, a następnie, przesyła do odbiorcy (lub odbiorców) jesli są aktywni.

### Synchronizacja wiadomości
Proces pobrania wcześniej wysłanych wiadomości.

#### Użytkownik
Użytkownik requestuje historię, dołączając swój token sesji wiadomości z danego okresu czasu pomiędzy odpowiednimy użytkownikami.
#### Server
Server waliduje token, a następnie w przypadku powodzenia odpowiada listą wiadomości.


## Struktura zapytań do servera json
Jak wygląda struktura zapytań do servera.<br>
Generalną zasadą jest to, że cała komunikacja przesyłana pomiędzy klientem a serverem powinna być wykonywana za pomocą zapytań json. Każde z zapytań ma taki sam szkielet,
składający się z pól: timestamp (zawierającego DateTime w standardzie UTC do ISO 8601, bez mikrosekund),
command (zawierającego nazwę komendy),
oraz payload (które różne pola w zależności od użytej komendy). <br>
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "command": "some_command",
  "payload": {
    ...
  }
}
```
### register
Uwaga, klient powinienen oczekiwać na potwierdzenie z serwera.
(czy np. nick już nie jest zajęty)
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "command": "register",
  "payload": {
    "name": "some-username",
    "pub": "user-rsa-public-key"
  }
}
```
### login
Uwaga, do pełnego zalogowania urzytkownik musi przesłać dwa requesty, request logowania,
oraz weryfikacje tożsamości.

request logowania:
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "command": "login",
  "payload": {
    "name": "some-username"
  }
}
```

weryfikacja tożsamości:
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "command": "login-verif",
  "payload": {
    "name": "some-username",
    "token-sig": "signature-of-token-in-hex"
  }
}
```


### logout
```json
{
  "imestamp":"2025-04-12T16:17:07+02:00",
  "command": "logout",
  "payload": {}
}
```
### message
```json
{
  "timestamp":"2025-04-12T16:17:07+02:00",
  "command": "message",
  "payload": {
    "from": "sender",
    "to": ["receiver1", "receiver2"],
    "aes": "aes-encrypted-with-rsa",
    "msg_cont": "message-encrypted-with-aes"
  }
}
```
Struktura wysłania wiadomości, zawiera pola które zawiera baza oraz w polu payload dodatkowo:
- from - od którego użytkownika idzie wiadomość. string
- to - do którego użytkownika/ów idzie wiadomość. tablica użytkowników (w przypadku jednego odbiorcy zawiera w sobie tylko jednego odbiorcę)
- aes - zaszyfrowany kluczem publiczym klucz do AES. string
- msg_cont - pole z treścią wiadomości zaszyfrowaną kluczem do AES. string

### online-list
listowanie użytkowników online
```json
{
  "timestamp":"2025-04-12T16:17:07+02:00",
  "command": "online-list",
  "payload": {}
}
```
### sync
komenda pozwalająca na pobranie wiadomości [TODO]
```json
{
  "timestamp":"2025-04-12T16:17:07+02:00",
  "command": "sync",
  "payload": {
    "from": "2025-04-10T16:20:47+02:00",
    "to": "2025-04-12T16:17:07+02:00",
    "participants": ["user1"]
  }
}
```
- from - przediał od kied dla wiadomości. String. DateTime UTC ISO 8601
- to - przedział do kied dla wiadomości. String. DateTime UTC ISO 8601
- participants - użytkownicy uczestniczący w konwersacji, bez podawania nadawcy requesta. Arr of str.

UWAGA! Zaleca się mieć na uwadze duży rozmiar odpowiedzi, przy wykonywaniu zapytania  
UWAGA! Powinno się ustawić jak najmniejszy możliwy zakres by zbytnio nie obciążyć serwera.



## Struktura odpowiedzi servera json
Struktura odpowiedzi servera na zapytania klienta.  
Header zawiera pole timestamp w formacie UTC ISO 8601, 
pole type odpowiadające za determinowanie rodzaju odpowiedz, 
oraz pole response, zawierające charakterystyczne wartości dla każdego z typów.

```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "type": "some_response",
  "response": {
    ...
  }
}
```
### error
Odpowiedź zawierająca informacje na temat błędu który zaszedł. Na przykład o niepowodzeniu próbie rejestracji.
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "type": "error",
  "response": {
    "message": "generic error message"
  }
}
```
### ack
Odpowiedź typu acknowledge, jest używana do potwierdzenia operacji, wychodzących od użytkownika. Na przykład o powodzeniu zalogowania.
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "type": "ack",
  "response": {
    "message": "generic ack message"
  }
}
```
### login
Odpowiedź serwera na [request logowania](https://github.com/Ochrona-Systemow-Operacyjnych-Grupa-2/notatka/tree/main?tab=readme-ov-file#login)
wysłany przez użytkownika. 
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "type": "login",
  "response": {
    "name": "some-username",
    "token-enc": "encrypted-token-in-hex"
  }
}
```
Pola reprezentują:
- name - powtórzona nazwa użytkownika.
- token-enc - losowy token, wygenerowany przez serwer, a następnie zaszyfowany za pomocą klucza publicznego użytkownika.

### sync
Odpowiedź, zawierająca zrequestowane wiadomości od urzytkownika.
```json
{
  "timestamp": "2025-04-12T16:17:07+02:00",
  "type": "sync",
  "response": {
    "from": "2025-04-10T16:20:47+02:00",
    "to": "2025-04-12T16:17:07+02:00",
    "participants": ["user1"],
    "payloads": [
      {
        "timestamp": "2025-04-10T16:20:47+02:00",
        "from": "sender",
        "to": ["receiver"],
        "aes": "aes-key",
        "msg_cont": "encrypted-msg"
      },
      {
        "timestamp": "2025-04-10T16:21:39+02:00",
        "from": "receiver",
        "to": ["sender"],
        "aes": "aes-key",
        "msg_cont": "encrypted-msg"
      },
      {
        ...
      }
    ]
  }
}
```
Poza headerem, odpowiedź zawiera pola:
- from - górny zakres wiadomości
- to - dolny zakres wiadomości
- participants - tablica nazw urzytkowników uczestniczących w rozmowie. (bez nazwy requestującego)
- payloads - tablica zawierająca główną odpowiedź o danych polach:
  - timestamp - data+czas odebrania wiadomości
  - from - nadawca wiadomości
  - to - tablica odbiorców wiadomości
  - aes - zaszyfrowany klucz aes
  - msg_cont - treść wiadomości zaszyfrowana kluczem aes

### online-list
Lista użytkowników online
```json
{
  "timestamp":"2025-04-12T16:17:07+02:00",
  "type": "online-list",
  "response": {
    "users": [
      {"name": "user1", "pub": "pub-key"},
      {"name": "user2", "pub": "pub-key"}
    ]
  }
}
```

## Struktura baz danych na serverze
bazy danych dotyczą przechowywania informacji o użytkownikach i wysłanych wiadomościach
### historia wiadomości
baza danych zawierająca wiadomości 

| msg_history |
| :---------: |
| id |
| timestamp |
| sender |
| receivers |
| msg |
| aes |

- id - id w bazie danych
- timestamp - data + godzina odebrania przez serwer wiadomośći
- sender - nadawca wiadomości
- receivers - odbiorcy wiadomości
- msg - zaszyfrowana treść wiadomości (za pomocą AES)
- aes - klucz AES do wiadmości zaszyfrowany przez RSA

### użytkownicy
baza danych zawierająca informacje o użytkownikach
| users | 
| :---------: |
| name | 
| pub | 

- name - nazwa użytkownika.
- pub - klucz publiczny z pary RSA użytkownika.

## niezmieniona oryginalna wersja (dodane tylko akapity)
  User i Serwer, każdy user musi się zarejestrować, rejestracja opisana na początku, User generuje sobie RSA (klucz publiczny i prywatny) wysyła na serwer publiczne RSA + nazwa użytkownika i serwer to zapisuje. Serwer sprawdza czy nazwa użytkownika jest zajęta, jeśli jest zajęta to wysyła do użytkownika, że rejestracja się nie powiodła, jeżeli się powiodła to wysyła informację zwrotną do użytkownika i użytkownik wie że istnieje w bazie.

  Nowy użytkownik musi się zalogować, żeby utwiorzyć sesję z której będzie korzystał, użytkownik wysyła do serwera request, żeby się zalogować, następnie serwer odsyła do użytkownika jego kluczem publicznym zaszyfrowany token w RSA i użytkownik odsyła do serwera token zaszyfrowany swoim kluczem prywatnym. Serwer sprawdza czy token jest taki sam, jeśli wszystko się zgadza serwer wysyła potwierdzenie, że jest logowanie. Jeśli jesteśmy zalogowani trafiamy do cache, z naszym tokenem i socketem.

  Wysyłanie wiadomości odbywa się w taki sposób, że klient tworzy JSON (from, to, wiadomość zaszyfrowana AESEM, timestamp wiadomości, losowy AES zaszyfrowany RSA publicznym odbiorcy). Przed wysłaniem wiadomości prosimy o klucz publiczny odbiorcy (użytkownika z pola to: ). Serwer sprawdza czy user jest online, jeśli jest to wysyła mu dalej tą wiadomość, w formacie w jakim dostał od użytkownika i zapisuje ta wiadomość w bazie danych. User2 odbiera wiadomość i widzi, że to jest do niego, bierze swój prywatny klucz rozszyfrowuje AES, i rozszyfrowuje wiadomość.
  
  Logout to User wysyła flagę do serwera (wylogowywuje się), serwer usuwa cię z Cache przy wylogowaniu. SYNC – user wysyła request do serwera o synchronizację wiadomości, które są do niego (od kiedy do kiedy, user token,) Serwer zwraca MSG history, które spełniają te wartości wysłane przez usera.
