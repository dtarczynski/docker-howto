# Docker — ściąga

Krótki przewodnik po pracy z Dockerem: najpierw pojęcia, potem budowa i uruchomienie obrazu, potem polecenia używane na co dzień. Przykłady dotyczą aplikacji ASP.NET Core `Codmatic.Api`, pakowanej jako obraz `dtarczynski/simpleapi`. Port na hoście zostaje **8989**.

## Spis treści

- [Pojęcia](#pojęcia)
- [Podstawowy przebieg](#podstawowy-przebieg)
- [Codzienne polecenia](#codzienne-polecenia)
- [Scenariusze](#scenariusze)
- [Przydatne linki](#przydatne-linki)

## Pojęcia

**Obraz (image)** to szablon tylko do odczytu: system plików, runtime i polecenie startowe. Ten sam obraz uruchamiasz wielokrotnie.

**Kontener** to działający egzemplarz obrazu. Ma własny proces, zapisywalną warstwę plików i sieć. Usunięcie kontenera zostawia obraz na dysku. Zmiany zrobione w kontenerze znikają razem z nim, chyba że zapiszesz je jako nowy obraz (`docker commit`) albo wynikają z Dockerfile.

**Dockerfile** to plik z instrukcjami, z których `docker build` składa obraz. Leży w katalogu projektu i nazywa się dokładnie `Dockerfile`.

**Rejestr (registry)** przechowuje obrazy i serwuje je innym maszynom. `docker pull` pobiera obraz, `docker push` wysyła. Publiczny rejestr to Docker Hub. Prywatny przykład dalej w tekście to Azure Container Registry pod adresem `hubdoc.azurecr.io`.

Nazwa obrazu ma postać `[rejestr/]nazwa[:tag]`. Bez hosta rejestru Docker przyjmuje Docker Hub. Bez tagu przyjmuje `latest`. `dtarczynski/simpleapi:latest` to repozytorium `simpleapi` użytkownika `dtarczynski` i tag `latest`.

## Podstawowy przebieg

### Dockerfile

Obraz `microsoft/aspnetcore:2.0` jest wycofany. Aktualny odpowiednik to `mcr.microsoft.com/dotnet/aspnet` z Microsoft Artifact Registry. Tag `10.0` to bieżące wydanie LTS (poprawki w ramach .NET 10). Tag `11.0` to bieżące wydanie w zwykłym okresie wsparcia.

W Dockerfile komentarz zaczyna się od `#` i zajmuje całą linię. Znak `#` w środku linii BuildKit traktuje jako argument instrukcji, więc komentarz przy `FROM` stoi w osobnych liniach nad nią.

Od .NET 8 ten obraz nasłuchuje na porcie **8080**. Obraz `microsoft/aspnetcore:2.0` nasłuchiwał na **80**, stąd dawne mapowanie `8989:80`. `EXPOSE` tylko dokumentuje port. Na zewnątrz wystawia go flaga `-p` przy `docker run`.

Katalog `bin/Release/net10.0/publish` to wynik publikacji projektu o docelowym frameworku `net10.0`. Nazwa zestawu zostaje `Codmatic.Api.dll`.

```bash
dotnet publish -c Release
```

Uruchom w katalogu projektu, zanim zbudujesz obraz. `-c Release` wybiera konfigurację Release. Do obrazu wchodzi opublikowany wynik, bez SDK.

```dockerfile
# Runtime ASP.NET Core, bez SDK.
# microsoft/aspnetcore:2.0 jest wycofany.
# Aktualny odpowiednik: mcr.microsoft.com/dotnet/aspnet, tag LTS 10.0.
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
EXPOSE 8080
COPY bin/Release/net10.0/publish .
ENTRYPOINT ["dotnet", "Codmatic.Api.dll"]
```

`WORKDIR /app` ustawia katalog roboczy kolejnych instrukcji i proces startowy. `COPY` bierze katalog publikacji z kontekstu budowania i kładzie jego zawartość w `/app`. `ENTRYPOINT` uruchamia aplikację przy starcie kontenera.

### Budowanie obrazu

Polecenie odpal w katalogu, w którym leżą `Dockerfile` i `bin/Release/net10.0/publish`.

```bash
docker build -t dtarczynski/simpleapi:latest .
```

`-t` (`--tag`) nadaje obrazowi nazwę. Tu nazwa to `dtarczynski/simpleapi`, tag to `latest`. Kropka na końcu to kontekst budowania: bieżący katalog, wysyłany do demona. `docker build` jest skrótem od `docker buildx build`. Brakujący obraz bazowy zostanie pobrany przy budowaniu.

### Lista obrazów

```bash
docker image ls
```

Pokazuje lokalne obrazy: repozytorium, tag, ID i rozmiar. To samo wypisuje `docker images`. Po budowaniu na liście jest `dtarczynski/simpleapi`.

### Uruchomienie kontenera

```bash
docker run --rm -it -p 8989:8080 dtarczynski/simpleapi
```

Tworzy kontener z obrazu i uruchamia `ENTRYPOINT`. Aplikacja jest pod adresem `http://localhost:8989`.

- `--rm` usuwa kontener (i jego anonimowe wolumeny) po zakończeniu procesu. Obraz zostaje.
- `-i` (`--interactive`) zostawia otwarte standardowe wejście.
- `-t` (`--tty`) przydziela pseudo-terminal. Razem `-it` dają sesję w terminalu. Ctrl+C wysyła sygnał do procesu w kontenerze.
- `-p` (`--publish`) mapuje porty w kolejności `port_hosta:port_kontenera`. `8989` to port na twojej maszynie (jak w dotychczasowym przykładzie), `8080` to port, na którym nasłuchuje bieżący obraz ASP.NET Core.
- Ostatni argument to obraz. Pominięty tag oznacza `latest`, czyli obraz z poprzedniego `docker build`.

Przy `microsoft/aspnetcore:2.0` to samo uruchomienie używało `-p 8989:80`. Z obecnym obrazem ruch na porcie 80 w kontenerze nie trafia w aplikację. Powrót na port 80 zmienną `ASPNETCORE_HTTP_PORTS=80` na tym obrazie zwykle się nie udaje: proces działa jako użytkownik inny niż root, a porty poniżej 1024 są wtedy niedostępne.

W tle, bez zajmowania terminala:

```bash
docker run --rm -d -p 8989:8080 dtarczynski/simpleapi
```

`-d` (`--detach`) startuje kontener w tle i wypisuje jego ID. W połączeniu z `--rm` kontener jest usuwany, gdy proces się skończy. Podgląd wyjścia daje `docker logs`.

## Codzienne polecenia

Identyfikator kontenera to ID albo nazwa z `docker ps`. Wystarczy unikalny początek ID. W przykładach `2db` i `6cf52a3b570b` to takie początki.

### Start i stop

```bash
docker start 2db
```

Wznawia zatrzymany kontener. `2db` to początek jego ID. Kontener utworzony z `--rm` po zakończeniu już nie istnieje — wtedy ponownie `docker run`.

```bash
docker stop 2db
```

Zatrzymuje jeden działający kontener. Proces dostaje SIGTERM, a po czasie oczekiwania SIGKILL. Kontener zostaje na liście zatrzymanych.

```bash
docker stop $(docker ps -q)
```

Zatrzymuje wszystkie działające kontenery. `docker ps -q` wypisuje same ID działających kontenerów, a `$(...)` wstawia je jako argumenty `docker stop`. Zatrzymanych kontenerów to polecenie nie dotyczy. Gdy nic nie działa, lista jest pusta i `docker stop` zgłasza brak argumentu.

### Lista kontenerów

```bash
docker ps
```

Lista działających kontenerów: ID, obraz, status, porty, nazwa. Skrót od `docker container ls`.

```bash
docker ps --all
```

Działające i zatrzymane. `--all` i krótsze `-a` znaczą to samo.

### Logi

```bash
docker logs 6cf52a3b570b
```

Wypisuje stdout i stderr kontenera. Użyj tego, gdy proces od razu spadł albo gdy kontener działa w tle (`-d`). `6cf52a3b570b` to ID albo jego unikalny początek.

```bash
docker logs -f 6cf52a3b570b
```

`-f` (`--follow`) zostawia strumień otwarty i dopisuje nowe linie. Ctrl+C kończy podgląd i zostawia kontener w spokoju.

### Pobranie obrazu

```bash
docker pull mcr.microsoft.com/dotnet/aspnet:10.0
```

Pobiera obraz z rejestru na dysk lokalny. Ten sam obraz jest bazą Dockerfile powyżej. `docker build` i `docker run` też pobierają brakujący obraz. `pull` przydaje się, gdy chcesz go mieć zawczasu albo odświeżyć wskazany tag. Bez `:tag` Docker bierze `latest`.

### Sprzątanie

Polecenia poniżej pytają o potwierdzenie. Flaga `-f` (`--force`) pomija pytanie.

```bash
docker container prune
```

Usuwa zatrzymane kontenery. Działające zostają. Uruchom, gdy po testach została lista kontenerów w stanie Exited.

```bash
docker image prune
```

Usuwa obrazy wiszące (dangling): bez tagu i nieużywane przez żaden kontener. Takie obrazy zostają po kolejnym `docker build` tego samego tagu, gdy tag przechodzi na nowy ID.

```bash
docker image prune --all
```

`--all` (krótko `-a`) usuwa dodatkowo otagowane obrazy, których nie używa żaden kontener. Sięga też po obrazy, które chcesz jeszcze kiedyś uruchomić, więc czytaj pytanie przed potwierdzeniem.

## Scenariusze

### Zapis zmian z działającego kontenera

Gdy w środku kontenera zmieniłeś pliki albo konfigurację i chcesz z tego nowy obraz:

```bash
docker commit container_id hubdoc.azurecr.io/myapp/database:latest
```

- `container_id` — ID albo nazwa z `docker ps`, na przykład `6cf52a3b570b`.
- `hubdoc.azurecr.io` — serwer logowania prywatnego rejestru, tu Azure Container Registry.
- `myapp/database` — ścieżka repozytorium wewnątrz rejestru.
- `latest` — tag nowego obrazu.

Polecenie zapisuje zmiany w systemie plików kontenera. Danych z zamontowanych wolumenów nie obejmuje. Na czas zapisu kontener jest wstrzymywany. Obraz ląduje lokalnie. Do rejestru wysyła go dopiero `docker push`. Powtarzalne budowanie aplikacji trzymaj w Dockerfile. `commit` zostaw do jednorazowego zrzutu stanu.

### Wysłanie obrazu do prywatnego rejestru

Poniżej ten sam rejestr: `hubdoc.azurecr.io`. Docker pyta o użytkownika i hasło albo token.

```bash
docker login hubdoc.azurecr.io
```

Loguje klienta Dockera do rejestru podanego jako argument. Adres to sam host, bez `https://` i bez ścieżki repozytorium. Po zalogowaniu `docker push` i `docker pull` na ten host używają zapisanych poświadczeń.

W Azure zwykle loguje się przez Azure CLI. Nazwa zasobu to `hubdoc`, bez domeny `.azurecr.io`.

```bash
az login
```

```bash
az acr login --name hubdoc
```

`az login` otwiera logowanie do subskrypcji. `az acr login --name` przyjmuje nazwę zasobu rejestru i wpisuje token do konfiguracji Dockera.

Obraz zbudowany jako `dtarczynski/simpleapi:latest` ma nazwę z Docker Hub. Prywatny rejestr wymaga nazwy z jego hostem. `docker tag` dokłada taką nazwę do tego samego obrazu.

```bash
docker tag dtarczynski/simpleapi:latest hubdoc.azurecr.io/myapp/simpleapi:latest
```

Pierwszy argument to obraz lokalny (`nazwa:tag`). Drugi to nowa nazwa: host rejestru, ścieżka `myapp/simpleapi` i tag `latest`. Warstwy zostają jedne.

```bash
docker push hubdoc.azurecr.io/myapp/simpleapi:latest
```

Wysyła do rejestru obraz wskazany pełną nazwą. Rejestr bierze się z hosta w nazwie. Tag pominięty w nazwie oznacza `latest`.

Obraz z `docker commit` jest już nazwany adresem rejestru, więc idzie bez ponownego tagowania:

```bash
docker push hubdoc.azurecr.io/myapp/database:latest
```

## Przydatne linki

- [Ściąga Docker CLI (PDF)](https://docs.docker.com/get-started/docker_cheatsheet.pdf) — oficjalna, linkowana też ze strony [Educational resources](https://docs.docker.com/get-started/resources/).
- [Dokumentacja Docker CLI](https://docs.docker.com/reference/cli/docker/)
- [Składnia Dockerfile](https://docs.docker.com/reference/dockerfile/)
- [Obrazy .NET w kontenerach](https://learn.microsoft.com/en-us/dotnet/core/docker/introduction)
- [Zmiana domyślnego portu ASP.NET Core z 80 na 8080](https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/8.0/aspnet-port)
- [Azure Container Registry: push i pull przez Docker CLI](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli) — dotychczasowy adres `docs.microsoft.com` przekierowuje tutaj.
- [Starsza ściąga ZeroTurnaround / JRebel (PDF)](https://files.jrebel.com/pdf/zt_docker_cheat_sheet.pdf) — plik spod `http://files.zeroturnaround.com/pdf/zt_docker_cheat_sheet.pdf` nadal się pobiera, po przekierowaniu na JRebel. Opisuje starsze polecenia, w tym Docker Machine.
