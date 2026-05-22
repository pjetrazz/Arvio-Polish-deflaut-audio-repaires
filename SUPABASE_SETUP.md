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

## Następny krok przy wznowieniu
- Zapytać użytkownika, na którym kroku jest / czy są błędy z `db push` lub `functions deploy`.
- Opcjonalnie: zweryfikować spójność migracji i `cloud-auth-email` pod rejestrację rodziny.
- Przy PR: w opisie zmiany „None" dopisać wymóg skonfigurowanego addonu (PL źródła).
