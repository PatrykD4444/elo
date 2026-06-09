## Pytanie 1.

> ***Opisz, jak aplikacja rozróżnia użytkowników o różnych uprawnieniach. Wskaż przykład sytuacji, w której jedna osoba może wykonać daną akcję, a inna nie powinna mieć do niej dostępu. Wyjaśnij, jak projekt to kontroluje.***

Mam trzy rodzaje użytkowników: gościa, który nie jest zalogowany, zwykłego zalogowanego użytkownika i administratora. To, czy ktoś jest adminem, trzymam jako zwykłą kolumnę `is_admin` w tabeli users, a w modelu User mam do tego metodę:

**Plik: app/Models/User.php (linie 31–35) — sprawdzenie, czy użytkownik jest administratorem**

```php
// Czy to administrator
public function isAdmin(): bool
{
    return $this->is_admin === true;
}
```

Uprawnienia sprawdzam w dwóch miejscach. Cały panel admina chroni mój middleware — jak ktoś nie jest adminem, dostaje 403:

**Plik: app/Http/Middleware/EnsureUserIsAdmin.php (linie 12–19) — ochrona całego panelu admina**

```php
// Wpuszcza tylko administratora, reszcie zwraca 403
public function handle(Request $request, Closure $next): Response
{
    if (! $request->user() || ! $request->user()->isAdmin()) {
        abort(403, 'Brak uprawnień. Ta sekcja jest dostępna tylko dla administratora.');
    }

    return $next($request);
}
```

A dostęp do konkretnego wydarzenia sprawdza polityka. Edytować można tylko swoje wydarzenie, ale admin każde:

**Plik: app/Policies/ActivityPolicy.php (linie 10–14) — kto może edytować dane wydarzenie**

```php
// Edycja: właściciel wydarzenia albo administrator
public function update(User $user, Activity $activity): bool
{
    return $user->isAdmin() || $activity->user_id === $user->id;
}
```

Przykład: Jan próbuje wejść w edycję wydarzenia Anny. Nie jest właścicielem i nie jest adminem, więc dostaje 403. Admin wejdzie bez problemu, bo `isAdmin()` go przepuszcza.

---

## Pytanie 2.

> ***Wybierz jedną konkretną akcję w aplikacji (np. dodanie, edycję, usunięcie lub wyświetlenie zasobu). Opisz krok po kroku drogę, jaką przebywają dane od momentu wywołania akcji w interfejsie użytkownika, aż do trwałego przetworzenia ich przez system i zwrócenia wyniku. W swoim opisie zidentyfikuj wszystkie warstwy i komponenty aplikacji, przez które przechodzi to żądanie, oraz wyjaśnij ich rolę w tym konkretnym procesie. Ważne: Swoją odpowiedź koniecznie udokumentuj, dołączając co najmniej 5 fragmentów kodu lub zrzutów ekranu reprezentujących poszczególne etapy tego procesu. Każdy załączony fragment krótko podpisz, wyjaśniając, jaki to plik i za który element przepływu odpowiada (np. przechwycenie danych z formularza, główna logika biznesowa, operacja zapisu do bazy danych, zwrócenie ostatecznego widoku/odpowiedzi).***

Opiszę dodanie wydarzenia, bo to przechodzi przez wszystkie warstwy.

Najpierw użytkownik wypełnia formularz w przeglądarce i klika „Zapisz”. Formularz idzie metodą POST i ma token CSRF, który chroni przed wysłaniem go z obcej strony.

**Fragment 1 – przechwycenie danych z formularza (resources/views/activities/_form.blade.php, linie 7–23)**

```blade
<form action="{{ $action }}" method="POST" class="space-y-5">
    @csrf
    @if ($method === 'PUT')
        @method('PUT')
    @endif

    {{-- Adres powrotu – żeby po zapisie wrócić tam, skąd otwarto formularz --}}
    @isset($redirectTo)
        <input type="hidden" name="redirect_to" value="{{ $redirectTo }}">
    @endisset

    <div>
        <label for="title" class="{{ $labelClass }}">Nazwa wydarzenia</label>
        <input type="text" name="title" id="title" value="{{ old('title', $activity->title ?? '') }}"
               placeholder="np. Wspólny mecz na Orliku" required class="{{ $inputClass }}">
        <x-input-error :messages="$errors->get('title')" class="mt-1.5" />
    </div>
```

Potem Laravel po adresie i metodzie HTTP znajduje pasującą trasę. Trasa zapisu jest w grupie auth, więc niezalogowany w ogóle tu nie wejdzie.

**Fragment 2 – dopasowanie trasy / routing (routes/web.php, linie 28–46)**

```php
Route::middleware('auth')->group(function () {
    // Moje wydarzenia (utworzone + te, do których dołączyłem)
    Route::get('/moje-wydarzenia', [ActivityController::class, 'mine'])->name('activities.mine');

    // Tworzenie / edycja / usuwanie WŁASNYCH wydarzeń
    Route::get('/wydarzenia/nowe', [ActivityController::class, 'create'])->name('activities.create');
    Route::post('/wydarzenia', [ActivityController::class, 'store'])->name('activities.store');
    Route::get('/wydarzenia/{activity}/edytuj', [ActivityController::class, 'edit'])->name('activities.edit');
    Route::put('/wydarzenia/{activity}', [ActivityController::class, 'update'])->name('activities.update');
    Route::delete('/wydarzenia/{activity}', [ActivityController::class, 'destroy'])->name('activities.destroy');

    // Dołączanie / rezygnacja
    Route::post('/wydarzenia/{activity}/dolacz', [ActivityController::class, 'join'])->name('activities.join');

    // Profil (Laravel Breeze)
    Route::get('/profile', [ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('/profile', [ProfileController::class, 'update'])->name('profile.update');
    Route::delete('/profile', [ProfileController::class, 'destroy'])->name('profile.destroy');
});
```

Żądanie trafia do metody store w kontrolerze. To główna logika akcji — sprawdzam, że to nie admin, oddaję dane do walidacji, a organizatora i status ustawiam sam, bo nie biorę ich z formularza.

**Fragment 3 – główna logika biznesowa (app/Http/Controllers/ActivityController.php, linie 82–100)**

```php
// Zapis nowego wydarzenia – trafia do akceptacji (status: pending)
public function store(Request $request)
{
    if (auth()->user()->isAdmin()) {
        return redirect()->route('admin.dashboard');
    }

    $data = $this->validateActivity($request);

    // user_id i status ustawiam ręcznie (nie z formularza)
    $activity = new Activity($data);
    $activity->user_id = auth()->id();
    $activity->status = Activity::STATUS_PENDING;
    $activity->save();

    return redirect()
        ->route('activities.mine')
        ->with('success', 'Wydarzenie zostało zgłoszone i czeka na akceptację administratora.');
}
```

Zanim coś zapiszę, dane idą przez walidację. Jak coś jest źle, Laravel sam zawraca do formularza i nic nie zapisuje.

**Fragment 4 – sprawdzenie poprawności danych / walidacja (app/Http/Controllers/ActivityController.php, linie 188–220)**

```php
// Walidacja danych wydarzenia (wspólna dla dodawania i edycji)
private function validateActivity(Request $request): array
{
    $validated = $request->validate([
        'title' => 'required|string|min:5|max:255',
        // dyscyplina z listy ALBO wpisana ręcznie
        'category_id' => 'nullable|required_without:new_category|exists:categories,id',
        'new_category' => 'nullable|string|max:100',
        'description' => 'nullable|string|max:1000',
        'location' => 'required|string|max:100',
        'place' => 'nullable|string|max:150',
        'skill_level' => 'required|in:Początkujący,Średni,Zaawansowany',
        'max_participants' => 'nullable|integer|min:1|max:1000',
        'date' => 'required|date|after:now',
    ], [
        'title.required' => 'Nazwa wydarzenia jest wymagana.',
        'title.min' => 'Nazwa musi mieć co najmniej 5 znaków.',
        'category_id.required_without' => 'Podaj dyscyplinę sportową.',
        'category_id.exists' => 'Wybrana dyscyplina nie istnieje.',
        'location.required' => 'Lokalizacja (miasto) jest wymagana.',
        'skill_level.in' => 'Wybierz poprawny poziom zaawansowania.',
        'date.required' => 'Data wydarzenia jest wymagana.',
        'date.after' => 'Data wydarzenia musi być z przyszłości.',
    ]);

    // Wpisana nowa dyscyplina – użyj istniejącej lub stwórz (bez duplikatów)
    if (! empty($validated['new_category'])) {
        $validated['category_id'] = Category::findOrCreateByName($validated['new_category'])->id;
    }
    unset($validated['new_category']);

    return $validated;
}
```

Sam zapis robi `save()` na modelu — to operacja zapisu do bazy. Model ma listę pól fillable, czyli tych, które wolno ustawić z formularza. `user_id` i `status` celowo tu nie ma, żeby nikt nie podstawił sobie cudzego właściciela albo nie zatwierdził wydarzenia z pominięciem moderacji.

**Fragment 5 – pola dozwolone przy zapisie do bazy (app/Models/Activity.php, linie 18–28)**

```php
// Pola, które można zapisać z formularza (user_id i status ustawiam w kontrolerze)
protected $fillable = [
    'category_id',
    'title',
    'description',
    'location',
    'place',
    'skill_level',
    'max_participants',
    'date',
];
```

Na koniec robię przekierowanie na „Moje wydarzenia” z komunikatem w sesji, zamiast od razu zwracać widok (zwrócenie odpowiedzi). To wzorzec PRG, dzięki któremu odświeżenie strony nie zapisze wydarzenia drugi raz. Czyli żądanie przeszło przez formularz, routing, middleware auth, kontroler, walidację, model i bazę, a potem wróciło do widoku.

---

## Pytanie 3.

> ***Wskaż miejsce, w którym aplikacja sprawdza poprawność danych wpisanych przez użytkownika. Opisz, jakie dane są sprawdzane, jakie błędy mogą zostać wykryte i co zobaczy użytkownik, jeśli wpisze niepoprawne dane.***

Dane sprawdzam po stronie serwera, w kontrolerze ActivityController, w metodzie `validateActivity` (jej pełny kod jest wyżej w pytaniu 2, plik app/Http/Controllers/ActivityController.php, linie 188–220). Mam ją jedną i używam jej i przy dodawaniu, i przy edycji, żeby nie pisać reguł dwa razy.

Sprawdzam między innymi, czy nazwa jest podana i ma co najmniej 5 znaków, czy dyscyplina istnieje w bazie albo czy wpisano nową, czy podano miasto, czy poziom jest jednym z trzech dozwolonych, czy liczba miejsc to liczba w sensownym zakresie i czy data jest poprawna oraz z przyszłości. Dzięki temu wyłapię puste pole, za krótką nazwę, tekst wpisany w pole liczbowe albo datę z przeszłości.

Jak coś jest źle, Laravel nie zapisuje i zawraca do formularza, a pod danym polem pokazuje mój polski komunikat, na przykład „Data wydarzenia musi być z przyszłości”. Wpisane wcześniej dane nie znikają, bo w formularzu używam funkcji `old`. Komunikaty pokazuję wspólnym komponentem:

**Plik: resources/views/components/input-error.blade.php (linie 1–9) — wyświetlanie błędów pod polem**

```blade
@props(['messages'])

@if ($messages)
    <ul {{ $attributes->merge(['class' => 'text-sm text-red-600 space-y-1']) }}>
        @foreach ((array) $messages as $message)
            <li>{{ $message }}</li>
        @endforeach
    </ul>
@endif
```

---

## Pytanie 4.

> ***Wskaż przykład widoku, który jest dobrze zorganizowany. Opisz, dlaczego ten widok jest czytelny, czy unika powtarzania tych samych fragmentów oraz czy łatwo byłoby go zmienić lub rozbudować.***

Najlepiej zorganizowany jest u mnie formularz wydarzenia, czyli plik resources/views/activities/_form.blade.php. To jeden wspólny szablon, którego używam w czterech miejscach: dodawanie i edycja przez użytkownika, edycja przez admina i okno podglądu w panelu. Zamiast czterech formularzy mam jeden, do którego przekazuję parametry, np. adres zapisu, metodę i podpis przycisku.

Jest czytelny, bo pola są pogrupowane, a powtarzające się klasy stylów trzymam w zmiennych na górze pliku, więc wszystko wygląda spójnie i zmieniam to w jednym miejscu:

**Plik: resources/views/activities/_form.blade.php (linie 2–5) — wspólne klasy stylów w jednym miejscu**

```blade
@php
    $inputClass = 'w-full bg-slate-50 border border-slate-200 rounded-xl px-4 py-2.5 text-sm text-slate-800 placeholder-slate-400 focus:bg-white focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 transition';
    $labelClass = 'block text-sm font-semibold text-slate-700 mb-1.5';
@endphp
```

Nie powtarza kodu, bo logikę formularza mam raz, a błędy wszędzie pokazuję tym samym komponentem. Łatwo go też rozbudować — jak ostatnio musiałem dodać ukryte pole z adresem powrotu, dopisałem to jednym warunkiem i od razu działało we wszystkich miejscach, gdzie ten formularz jest używany:

**Plik: resources/views/activities/_form.blade.php (linie 13–16) — łatwe dodanie nowego pola**

```blade
{{-- Adres powrotu – żeby po zapisie wrócić tam, skąd otwarto formularz --}}
@isset($redirectTo)
    <input type="hidden" name="redirect_to" value="{{ $redirectTo }}">
@endisset
```

---

## Pytanie 5.

> ***Wskaż jedną funkcjonalność, której nie widać od razu w interfejsie, ale która jest ważna dla działania projektu. Opisz, gdzie znajduje się w projekcie i dlaczego jest potrzebna. Może to być np. przygotowanie danych, sprawdzanie uprawnień, filtrowanie wyników, zapisywanie informacji w bazie, obsługa błędów, porządkowanie danych albo automatyczne ustawianie wartości.***

Najważniejsze, czego nie widać wprost, to scope’y w modelu Activity i to, że część pól ustawiam sam, zamiast brać je z formularza. Na stronie głównej widać tylko listę wydarzeń, ale w tle każde zapytanie przechodzi przez `approved` i `upcoming`, więc gość nigdy nie zobaczy wydarzeń oczekujących, odrzuconych ani tych z przeszłości:

**Plik: app/Models/Activity.php (linie 56–72) — filtrowanie wyników (scope zapytań)**

```php
// --- Skróty do zapytań (scope), np. Activity::approved() ---

public function scopeApproved(Builder $query): Builder
{
    return $query->where('status', self::STATUS_APPROVED);
}

public function scopePending(Builder $query): Builder
{
    return $query->where('status', self::STATUS_PENDING);
}

// Tylko nadchodzące (data w przyszłości)
public function scopeUpcoming(Builder $query): Builder
{
    return $query->where('date', '>=', now());
}
```

Druga rzecz to automatyczne ustawianie organizatora i statusu przy zapisie (widać to w metodzie store w pytaniu 2). Użytkownik nie ma tych pól w formularzu, ustawiam je w kontrolerze. To ważne dla bezpieczeństwa, bo inaczej ktoś mógłby podstawić cudze konto jako właściciela albo od razu zatwierdzić sobie wydarzenie bez admina. Bez tego cała moderacja by nie działała.

---

## Pytanie 6.

> ***Opisz, w jaki sposób aplikacja rozpoznaje, co użytkownik chce zrobić (pobrać dane, edytować, usunąć, utworzyć). Odnieś się do adresu strony, rodzaju żądania, formularza, ścieżki w aplikacji oraz fragmentu kodu, który obsługuje daną akcję.***

Rozpoznaje to po adresie strony i metodzie HTTP, które dopasowuje do trasy w routes/web.php. Ten sam zasób obsługuję różnie zależnie od metody: GET na adres wydarzenia to wyświetlenie (metoda show), POST na /wydarzenia to utworzenie (store), PUT na konkretne wydarzenie to zapis edycji (update), a DELETE to usunięcie (destroy). Widać to w tych trasach:

**Plik: routes/web.php (linie 33–37) — różne akcje na tym samym zasobie wg metody HTTP**

```php
Route::get('/wydarzenia/nowe', [ActivityController::class, 'create'])->name('activities.create');
Route::post('/wydarzenia', [ActivityController::class, 'store'])->name('activities.store');
Route::get('/wydarzenia/{activity}/edytuj', [ActivityController::class, 'edit'])->name('activities.edit');
Route::put('/wydarzenia/{activity}', [ActivityController::class, 'update'])->name('activities.update');
Route::delete('/wydarzenia/{activity}', [ActivityController::class, 'destroy'])->name('activities.destroy');
```

Formularz HTML umie wysłać tylko GET albo POST, więc przy edycji i usuwaniu używam w formularzu dyrektywy `@method('PUT')` albo `@method('DELETE')`, a Laravel czyta to ukryte pole i kieruje żądanie do właściwej metody kontrolera. Dodatkowo dzięki route model binding parametr `{activity}` w adresie sprawia, że Laravel sam pobiera odpowiednie wydarzenie z bazy po jego ID i podaje mi je do metody jako gotowy obiekt.

---

## Pytanie 7.

> ***Opisz jeden przykład powiązania danych w projekcie. Wybierz dwa typy danych, które są ze sobą powiązane, np. użytkownik i jego wpisy, kategoria i produkty, zamówienie i pozycje zamówienia. Wyjaśnij, jak to powiązanie jest widoczne w bazie danych, kodzie i widoku.***

Biorę powiązanie użytkownik i jego wydarzenia, czyli jeden do wielu. Jeden użytkownik może mieć wiele wydarzeń, a każde wydarzenie ma jednego organizatora.

W bazie widać to jako kolumnę `user_id` w tabeli activities, która jest kluczem obcym wskazującym na `users.id`. W kodzie opisuję to z obu stron:

**Plik: app/Models/Activity.php (linie 44–48) — strona „wydarzenie należy do użytkownika”**

```php
// Organizator (twórca wydarzenia)
public function organizer()
{
    return $this->belongsTo(User::class, 'user_id');
}
```

**Plik: app/Models/User.php (linie 37–41) — strona „użytkownik ma wiele wydarzeń”**

```php
// Wydarzenia utworzone przez użytkownika
public function createdActivities()
{
    return $this->hasMany(Activity::class, 'user_id');
}
```

W widoku korzystam z tego, pokazując imię organizatora przez `$activity->organizer->name`, a w „Moich wydarzeniach” listuję wszystkie wydarzenia danego użytkownika. Mam też trudniejszą relację wiele do wielu, czyli uczestnicy zapisani na wydarzenia, opartą o tabelę pośrednią activity_user:

**Plik: app/Models/Activity.php (linie 50–54) — relacja wiele-do-wielu (uczestnicy)**

```php
// Zapisani uczestnicy
public function participants()
{
    return $this->belongsToMany(User::class)->withTimestamps();
}
```

---

## Pytanie 8.

> ***Opisz, jak aplikacja reaguje na nietypową sytuację. Może to być np. próba wejścia na nieistniejący zasób, wpisanie błędnych danych, brak wymaganych uprawnień, brak pliku, pusta lista wyników albo błąd zapisu. Wskaż, co zobaczy użytkownik i co dzieje się w kodzie.***

Aplikacja w kilku miejscach radzi sobie z dziwnymi sytuacjami. Pierwsza to próba wejścia na wydarzenie, którego nie powinno się widzieć. Jak gość zna ID wydarzenia, które dopiero czeka na akceptację, i wejdzie na jego stronę, to w metodzie show sprawdzam, że nie jest zaakceptowane, i zwracam 404. Daję 404 zamiast 403 specjalnie, żeby nie zdradzać, że taki zasób w ogóle istnieje:

**Plik: app/Http/Controllers/ActivityController.php (linie 57–68) — ukrycie niezaakceptowanych wydarzeń**

```php
// Szczegóły wydarzenia (niezaakceptowane widzi tylko właściciel lub admin)
public function show(Activity $activity)
{
    if (! $activity->isApproved()) {
        $user = auth()->user();
        abort_unless($user && ($user->isAdmin() || $activity->user_id === $user->id), 404);
    }

    $activity->load(['category', 'organizer', 'participants']);

    return view('activities.show', compact('activity'));
}
```

Druga sytuacja to edycja cudzego wydarzenia — wtedy polityka nie przepuszcza żądania i użytkownik dostaje 403. Trzecia to dołączanie, gdy nie ma już miejsc albo termin minął — wtedy nie zapisuję uczestnika, tylko wracam z czerwonym komunikatem, np. „Brak wolnych miejsc na to wydarzenie”. Obsługuję też pustą listę — jak filtr nic nie znajdzie, w widoku mam `forelse` z sekcją `empty`, która pokazuje komunikat zamiast pustej strony. Wszędzie tam użytkownik dostaje normalny komunikat po polsku, a nie błąd serwera.

---

## Pytanie 9.

> ***Wskaż jedno miejsce w projekcie, które Twoim zdaniem powinno zostać poprawione technicznie. Opisz konkretny problem, jego wpływ na działanie lub rozwijanie projektu oraz zaproponuj sposób poprawy.***

Najbardziej warto poprawić walidację, którą trzymam w kontrolerze w metodzie `validateActivity` (plik app/Http/Controllers/ActivityController.php, linie 188–220). Działa dobrze, ale przez to kontroler robi dwie rzeczy naraz — sprawdza dane i obsługuje zapis. Im więcej reguł bym dokładał, tym dłuższy i mniej czytelny by się robił, no i samą walidację trudniej tak przetestować osobno.

Poprawiłbym to, przenosząc reguły do osobnej klasy Form Request, którą Laravel ma właśnie do tego. Wtedy kontroler dostawałby już sprawdzone dane, a reguły i komunikaty miałyby swój własny plik. To bardziej „laravelowy” sposób i kod byłby czystszy. Nie zrobiłem tego od razu, bo przy tej wielkości projektu prościej było mieć walidację w jednym miejscu, ale gdybym projekt rozwijał, zacząłbym właśnie od tego.

---

## Pytanie 10.

> ***Napisz 6–10 zdań, w których wyjaśnisz, dlaczego projekt zasługuje na wskazaną ocenę. Odwołaj się do konkretnych przykładów z wcześniejszych odpowiedzi, np. działania aplikacji, obsługi żądań, formularzy, walidacji, bazy danych, relacji, uprawnień, widoków, bezpieczeństwa, kompletności lub jakości technicznej.***

Uważam, że projekt zasługuje na 3.5, bo robi w pełni to, co jest wymagane na trójkę, i dokłada do tego sporo rzeczy z wyższego poziomu. Na trójkę mam ogólnodostępne zasoby i pełny CRUD wydarzeń. Do tego dodałem trzy role i podział obowiązków, gdzie użytkownik zarządza swoimi wydarzeniami, a admin moderuje i może zarządzać cudzymi. Uprawnienia sprawdzam w dwóch miejscach, bo middleware chroni cały panel admina, a polityka pilnuje dostępu do pojedynczego wydarzenia. Dane z formularzy są walidowane po stronie serwera z czytelnymi komunikatami, a po zapisie robię przekierowanie, żeby nie dało się zapisać czegoś dwa razy. W bazie mam dwie relacje, jeden do wielu i wiele do wielu. Zadbałem też o bezpieczeństwo, czyli tokeny CSRF, hashowanie haseł i korzystanie z Eloquent zamiast surowego SQL. Brakuje za to bardziej rozbudowanych rzeczy, jak powiadomienia mailowe czy oddzielne klasy walidacji, dlatego sam oceniam to na 3.5, a nie wyżej.

---

## Pytanie 11.

> ***Co należy poprawić w pierwszej kolejności, aby projekt zasługiwał na wyższą ocenę? Podaj jedną konkretną zmianę i wyjaśnij, dlaczego właśnie ona jest najważniejsza.***

Najpierw dodałbym powiadomienia e-mail o zmianie statusu wydarzenia, czyli wiadomość do użytkownika, gdy admin zaakceptuje albo odrzuci jego zgłoszenie. Teraz użytkownik musi sam wchodzić w „Moje wydarzenia”, żeby sprawdzić, co się stało, i to jest luka. Uważam, że to najważniejsze, bo zamyka cały proces moderacji, który jest u mnie najważniejszy — użytkownik od razu by wiedział, czy wydarzenie przeszło i z jakiego powodu ewentualnie je odrzucono. Zrobiłbym to klasą powiadomień Laravela, wysyłaną przy akceptacji i odrzuceniu w panelu admina. To prawdziwa nowa funkcja, a nie kosmetyka, więc najbardziej podniosłoby wartość projektu.

---

## Pytanie 12.

> ***Wyjaśnij, jak przygotowano strukturę bazy danych projektu. Wskaż przykład tabeli utworzonej w projekcie, opisz jej najważniejsze pola oraz wyjaśnij, skąd biorą się przykładowe lub początkowe dane, jeśli takie występują.***

Baza danych projektu działa na SQLite (plik database/database.sqlite). Strukturę tabel przygotowałem przy użyciu migracji Laravela, czyli plików w folderze database/migrations, które tworzą tabele krokami. Każda migracja opisuje jedną zmianę, na przykład utworzenie tabeli albo dodanie do niej kolumny, i odpalam je komendą `php artisan migrate`. Dzięki temu całą bazę da się odtworzyć od zera na innym komputerze.

Przykładem jest tabela `activities`, czyli wydarzenia. Tworzy ją ta migracja:

**Plik: database/migrations/2026_05_31_124703_create_activities_table.php (linie 14–28) — utworzenie tabeli wydarzeń**

```php
Schema::create('activities', function (Blueprint $table) {
    $table->id();
    
    // Relacja z tabelą categories (One-to-Many)
    $table->foreignId('category_id')->constrained()->cascadeOnDelete();
    
    // Pozostałe dane wydarzenia
    $table->string('title');
    $table->text('description')->nullable();
    $table->string('location'); // Miasto
    $table->string('skill_level'); // np. Początkujący, Średni
    $table->dateTime('date'); // Data i czas wydarzenia
    
    $table->timestamps();
});
```

Najważniejsze pola to `id` (klucz główny), `category_id` (klucz obcy do dyscypliny), `title` (nazwa wydarzenia), `location` (miasto), `date` (data i godzina) oraz `skill_level` (poziom). Później dołożyłem osobną migracją jeszcze `user_id`, czyli organizatora, i `status` (oczekuje, zaakceptowane, odrzucone), bo to doszło razem z rolami i moderacją:

**Plik: database/migrations/2026_06_06_120000_add_owner_and_status_to_activities_table.php (linie 14–20) — dodanie organizatora i statusu moderacji**

```php
Schema::table('activities', function (Blueprint $table) {
    // Twórca/organizator wydarzenia. Usunięcie konta usuwa też jego wydarzenia.
    $table->foreignId('user_id')->nullable()->after('id')->constrained()->cascadeOnDelete();

    // Status moderacji: pending (oczekuje), approved (na tablicy), rejected (odrzucone)
    $table->string('status')->default('pending')->after('date');
});
```

Przykładowe dane biorą się z seederów, czyli plików w database/seeders. Po wpisaniu `php artisan migrate:fresh --seed` tworzą się konta startowe (admin i dwóch użytkowników z hasłem `password`), lista dyscyplin oraz kilka przykładowych wydarzeń, część zaakceptowanych i część oczekujących, żeby od razu było widać, jak działa panel admina.

---

## Pytanie 13.

> ***Wyjaśnij, jak projekt zabezpiecza formularze przed niepożądanym lub przypadkowym wysłaniem danych z zewnątrz. Wskaż konkretny formularz i opisz, jaki mechanizm sprawia, że aplikacja rozpoznaje, czy żądanie pochodzi z poprawnej strony.***

Wszystkie moje formularze, które zmieniają dane, są zabezpieczone tokenem CSRF. W każdym formularzu jest dyrektywa `@csrf`, która dodaje ukryte pole z losowym tokenem przypisanym do sesji użytkownika. Przykład z formularza dodawania wydarzenia:

**Plik: resources/views/activities/_form.blade.php (linie 7–11) — token CSRF w formularzu**

```blade
<form action="{{ $action }}" method="POST" class="space-y-5">
    @csrf
    @if ($method === 'PUT')
        @method('PUT')
    @endif
```

Kiedy formularz jest wysyłany, Laravel sprawdza, czy token z formularza zgadza się z tym zapisanym w sesji. Jeśli ktoś spróbowałby podstawić formularz z innej strony, nie miałby poprawnego tokenu i żądanie zostanie odrzucone z błędem 419. Dzięki temu aplikacja rozpoznaje, że żądanie naprawdę przyszło z mojej strony, a nie z obcego miejsca. Robi to wbudowany mechanizm Laravela, więc nie muszę tego pisać ręcznie, wystarczy że dodaję `@csrf` w każdym formularzu zmieniającym dane.

---

## Pytanie 14.

> ***Wskaż po jednym przykładzie projektu z grupy, który oceniasz jako: mocniejszy od Twojego projektu, słabszy od Twojego projektu. Możesz wskazać autora projektu z imienia i nazwiska albo podać temat projektu, jeśli pozwala to jednoznacznie go rozpoznać. Przy każdym przykładzie krótko uzasadnij wybór, odnosząc się do konkretnych kryteriów, np. funkcjonalności, kompletności, jakości kodu, organizacji plików, pracy z bazą danych, obsługi formularzy, uprawnień, bezpieczeństwa, estetyki, działania aplikacji lub poziomu trudności. Nie oceniaj autora projektu — oceń wyłącznie sam projekt i jego wykonanie.***

Za mocniejszy od mojego uważam projekt (temat albo imię i nazwisko). Oceniam go wyżej, bo (np. ma płatności albo powiadomienia mailowe, albo więcej relacji w bazie, albo lepiej rozdzielony kod z osobnymi klasami walidacji). Wygrywa głównie kompletnością i wyższym poziomem trudności, bo robi rzeczy, których u siebie nie zaimplementowałem.

Za słabszy od mojego uważam projekt (temat albo imię i nazwisko). Oceniam go niżej, bo (np. ma tylko jedną rolę i podstawowy CRUD bez sprawdzania dostępu do pojedynczego rekordu, albo walidacja jest tylko po stronie przeglądarki, albo brakuje obsługi błędów i powtarza się kod w widokach). U mnie te rzeczy są zrobione, bo mam trzy role, polityki dostępu, walidację po stronie serwera i moderację, dlatego uważam swój projekt za bardziej kompletny. Oceniam przy tym samo wykonanie, a nie autora.
