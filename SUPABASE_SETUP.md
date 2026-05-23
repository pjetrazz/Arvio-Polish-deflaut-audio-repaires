# Arvio — notatki robocze (Supabase + Trakt + konta rodziny)

Stan na: 2026-05-22. Branch: `claude/github-access-request-jpjwv`.

## Kontekst / status zadań

1. **Polish language fix („None")** — ZROBIONE, wypchnięte (commit `a2c3e13`).
   - Dodana opcja **Default Audio → None**.
   - `None` = brak preferencji języka: autoplay bierze **górę listy z addonu** (jak Nuvio
     „Auto-play first source"), nie wymusza ścieżki audio, nie przesortowuje źródeł.
   - Ustawiony konkretny język → działa sortowanie language-first.
   - **Do opisu PR dopisać:** wymaga skonfigurowanego addonu, który podaje polskie
     źródła na górze (np. Torrentio z priorytetem PL + Torbox).
   - Pliki: `PlayerViewModel.kt` (resolvePreferredAudioLanguage, sortStreamsByQualityAndSize),
     `PlayerScreen.kt` (setPreferredAudioLanguage x2 + guard auto-selektu),
     `SettingsViewModel.kt` (lista opcji audio).

2. **Trakt „activation failed (403)"** — DIAGNOZA GOTOWA, czeka na konfigurację backendu.
   - To NIE jest bug w kodzie aplikacji. Klient działa poprawnie.
   - Ruch Trakt idzie przez Edge Function `supabase/functions/trakt-proxy/index.ts`,
     która wstrzykuje `trakt-api-key` z sekretu serwerowego `TRAKT_CLIENT_ID`.
   - `403` może wyjść tylko z samego Trakt.tv = **invalid API key / unapproved app**
     (zły/cofnięty Client ID albo apka Trakt niezatwierdzona/usunięta, albo pusty klucz).
     (404 = funkcja niewdrożona, 401 = zły anon key, 429 = limit — tu jest 403.)
   - Sekrety builda wchodzą z GitHub Actions (`.github/workflows/build-apk.yml`):
     `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `TRAKT_CLIENT_ID`, `TRAKT_CLIENT_SECRET`, `TMDB_API_KEY`.
   - Sekrety SERWEROWE funkcji ustawia się osobno przez `supabase secrets set`.

3. **Konta rodziny (Supabase)** — DO ZROBIENIA po postawieniu backendu.
   - Dobra wiadomość: konta e-mail + wiele profili są JUŻ zaprogramowane.
     Funkcja `cloud-auth-email`, migracje tworzą tabele kont/profili.
     Po uruchomieniu backendu każdy z rodziny zakłada własne konto.
   - DO WERYFIKACJI: czy `cloud-auth-email` faktycznie obsługuje rejestrację
     (signup) tak jak chcemy dla rodziny.

## Stan danych użytkownika (z rozmowy)
- Ma świeże konto Supabase (free tier) — pusty projekt.
- Ma zarejestrowaną aplikację Trakt: Client ID + Secret.
- Używa Torrentio + Torbox jako scrapera.
- Build: V1.9.93-debug (sideload debug z GitHub Actions).

## Co jest w repo (wszystko gotowe do wdrożenia)
- Edge Functions (`supabase/functions/`): `trakt-proxy`, `tmdb-proxy`,
  `cloud-auth-email`, `cloud-auth-reset`, `tv-auth-start`, `tv-auth-status`,
  `tv-auth-approve`, `tv-auth-complete`.
- Migracje (`supabase/migrations/`): 6 plików (RLS/audit, tv-auth+account-sync,
  user_settings, watch_history+stream_pinning, profile_realtime_sync, profile_sync_state).
- `trakt-proxy` czyta env: `TRAKT_CLIENT_ID`, `TRAKT_CLIENT_SECRET`, `APP_ANON_KEY`.
- `APP_ANON_KEY` MUSI być identyczny z `SUPABASE_ANON_KEY` projektu.

## Procedura wdrożenia (jednorazowo)

```bash
# 1. CLI
npm i -g supabase
supabase login

# 2. Połącz projekt (ref: dashboard → Project Settings → General → Reference ID)
supabase link --project-ref TWOJ_PROJECT_REF

# 3. Schemat bazy (tabele kont/profili/historii/ustawień)
supabase db push

# 4. Sekrety serwerowe (naprawia Trakt 403). APP_ANON_KEY == anon key projektu (Settings → API)
supabase secrets set \
  TRAKT_CLIENT_ID=twoj_trakt_client_id \
  TRAKT_CLIENT_SECRET=twoj_trakt_secret \
  TMDB_API_KEY=twoj_tmdb_key \
  APP_ANON_KEY=twoj_supabase_anon_key

# 5. Wdróż funkcje
supabase functions deploy trakt-proxy tmdb-proxy cloud-auth-email cloud-auth-reset \
  tv-auth-start tv-auth-status tv-auth-approve tv-auth-complete
```

**6. GitHub repo → Settings → Secrets and variables → Actions:**
- `SUPABASE_URL` = `https://TWOJ_REF.supabase.co`
- `SUPABASE_ANON_KEY` = anon key
- `TRAKT_CLIENT_ID`, `TRAKT_CLIENT_SECRET`, `TMDB_API_KEY`

**7. Trakt (trakt.tv/oauth/applications):** Redirect URI = `urn:ietf:wg:oauth:2.0:oob`,
aplikacja zapisana/zatwierdzona.

**8. Odpal nowy build** (Actions → Build APK → Run workflow), zainstaluj, połącz Trakt,
przetestuj logowanie e-mail.

## WAŻNE: każdy ma swój Trakt (nie współdzielą konta)
- Trakt Client ID/Secret = tożsamość APLIKACJI, nie konto użytkownika.
  Wszyscy korzystają z tego samego Client ID (jak w każdym OAuth) — to bezpieczne.
- Każdy z rodziny robi „Connect Trakt" i loguje się SWOIM kontem na trakt.tv/activate,
  dostaje WŁASNY token. Historia/scrobble lecą na jego konto.
- Kod już trzyma tokeny per profil: `accessTokenKey() = profileManager.profileStringKey("trakt_access_token")`.
- Proxy używa: Twój Client ID (serwerowo) + token usera (nagłówek `x-user-token`).
- Wniosek: wpisujesz SWÓJ Client ID/Secret raz, działa dla całej rodziny, każdy ma swój Trakt.

## Instrukcja krok-po-kroku (na komputerze, nie telefonie; wymaga Node.js + git)

### 1. Dane z Supabase (app.supabase.com → projekt)
- Project Settings → General → Reference ID
- Project Settings → API → Project URL + anon public key
- Project Settings → Database → hasło do bazy (lub reset)

### 2. Pobierz repo
```bash
git clone -b claude/github-access-request-jpjwv https://github.com/pjetrazz/arvio-polish-deflaut-audio-repaires.git
cd arvio-polish-deflaut-audio-repaires
```

### 3. CLI + link
```bash
npm i -g supabase
supabase login
supabase link --project-ref TWOJ_REFERENCE_ID   # poprosi o hasło do bazy
```

### 4. Baza
```bash
supabase db push
```

### 5. Sekrety serwerowe (naprawia Trakt 403; APP_ANON_KEY == anon key)
```bash
supabase secrets set TRAKT_CLIENT_ID=...
supabase secrets set TRAKT_CLIENT_SECRET=...
supabase secrets set TMDB_API_KEY=...
supabase secrets set APP_ANON_KEY=...
```

### 6. Funkcje
```bash
supabase functions deploy trakt-proxy tmdb-proxy cloud-auth-email cloud-auth-reset tv-auth-start tv-auth-status tv-auth-approve tv-auth-complete
```

### 7. GitHub repo → Settings → Secrets and variables → Actions
SUPABASE_URL, SUPABASE_ANON_KEY, TRAKT_CLIENT_ID, TRAKT_CLIENT_SECRET, TMDB_API_KEY

### 8. Trakt app (trakt.tv/oauth/applications)
Redirect URI = `urn:ietf:wg:oauth:2.0:oob`, zapisz.

### 9. Build + test
Actions → Build APK → Run workflow → zainstaluj APK → Settings → Trakt → Connect (kod zamiast 403),
Cloud Account → Sign In.

### 10. Rodzina
Każdy robi Connect Trakt swoim kontem; Client ID wspólny, tokeny/historia osobne.

## WYBRANA DROGA: Integracja Supabase↔GitHub
Użytkownik ma połączony Supabase z GitHubem i wybrał poleganie na integracji.
- Integracja sama odpala migracje (i funkcje, jeśli plan pozwala) na push.
- NIE ustawia sekretów funkcji → trzeba je wpisać ręcznie w dashboardzie
  (Edge Functions → Secrets): TRAKT_CLIENT_ID, TRAKT_CLIENT_SECRET, TMDB_API_KEY,
  APP_ANON_KEY (= anon key).
- Sekrety buildu APK i tak idą do GitHub: SUPABASE_URL, SUPABASE_ANON_KEY,
  TRAKT_CLIENT_ID, TRAKT_CLIENT_SECRET, TMDB_API_KEY.
- UWAGA free tier: auto-deploy przez integrację zwykle wymaga Branchingu (plan Pro).
  Jeśli tabele/funkcje się NIE pojawią → fallback: workflow `.github/workflows/deploy-supabase.yml`
  (wymaga dodatkowo sekretów SUPABASE_ACCESS_TOKEN, SUPABASE_PROJECT_REF, SUPABASE_DB_PASSWORD).
- Weryfikacja: Supabase → Database → Tables (czy są tabele) oraz Edge Functions (czy są funkcje).

## Następny krok przy wznowieniu
- Zapytać użytkownika, na którym kroku jest / czy są błędy z `db push` lub `functions deploy`.
- Opcjonalnie: zweryfikować spójność migracji i `cloud-auth-email` pod rejestrację rodziny.
- Przy PR: w opisie zmiany „None" dopisać wymóg skonfigurowanego addonu (PL źródła).
