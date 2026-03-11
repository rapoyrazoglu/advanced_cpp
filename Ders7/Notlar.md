# Ders 7 — Perfect Returning, `decltype(auto)` Tuzakları, Tag Dispatch ve User-Defined Literals

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 16 Nisan 2025
> **Kapsam:** Perfect returning (`decltype(auto)`), lambda'da perfect returning, `decltype(auto)` return type quiz (identifier vs expression kuralları, parantez tuzağı, ternary operator, member access xvalue), overload resolution'da universal reference önceliği, `std::ref` / `std::reference_wrapper`, tag dispatch tekniği (empty/tag class), user-defined literals (literal operator function, cooked/raw form, numeric literal operator template, string UDL C++20, standart kütüphane UDL'leri, strong typing)

---

## 1. Perfect Returning — `decltype(auto)` ile Geri Dönüş

### 1.1 Kavram

Perfect forwarding argümanların value category'sini koruyarak iletirken, **perfect returning** çağrılan fonksiyonun geri dönüş değerini çağırana **value category'sini koruyarak** döndürmeyi ifade eder.

```cpp
// Perfect returning — decltype(auto) kullanımı
template<typename F, typename... Args>
decltype(auto) wrapper(F&& f, Args&&... args) {
    return std::forward<F>(f)(std::forward<Args>(args)...);
}
```

`decltype(auto)` geri dönüş değeri türünü `return` ifadesine `decltype` kurallarını uygulayarak çıkarır:

| `return` İfadesi | `decltype` Kuralı | Çıkarılan Tür |
|---|---|---|
| identifier (`x`) | Declaration type | `T` (referans değil) |
| lvalue expression (`(x)`) | lvalue → `T&` | `T&` |
| xvalue expression | xvalue → `T&&` | `T&&` |
| prvalue expression | prvalue → `T` | `T` |

### 1.2 Lambda'da Perfect Returning

```cpp
// Lambda ile perfect returning — trailing return type gerekli
auto wrapper = []<typename... Args>(Args&&... args) -> decltype(auto) {
    return func(std::forward<Args>(args)...);
};
```

Trailing return type (`-> decltype(auto)`) yazılmazsa, lambda'nın geri dönüş değeri `auto` çıkarımıyla belirlenir — bu referansları düşürür ve perfect returning bozulur.

### 1.3 `if constexpr` ile Perfect Returning

```cpp
template<typename F, typename... Args>
decltype(auto) call_and_return(F&& f, Args&&... args) {
    decltype(auto) ret = std::forward<F>(f)(std::forward<Args>(args)...);

    if constexpr (std::is_rvalue_reference_v<decltype(ret)>) {
        return std::move(ret);   // rvalue referans → move ile döndür
    } else {
        return ret;              // lvalue ref veya value → doğrudan döndür
    }
}
```

> **Önemli Not:** `decltype(auto) ret = expr;` ile değişkeni yakalayıp sonra koşullu döndürmek, arada ek işlem yapmak gerektiğinde (logging, transformation vb.) kullanılan bir pattern'dir. Basit pass-through için doğrudan `return` ifadesi yeterlidir.

---

## 2. `decltype(auto)` Return Type — Quiz ve Tuzaklar

### 2.1 Identifier Kuralı — Güvenli

```cpp
decltype(auto) foo(int i) {
    return i;    // ✓ decltype(i) = int → geri dönüş: int
}
```

`i` bir identifier olduğu için `decltype` declaration type'ı döndürür: `int`. Sorun yok.

### 2.2 Parantez Tuzağı — **Tanımsız Davranış!**

```cpp
decltype(auto) foo(int x) {
    return (x);  // ❌ decltype((x)) = int& → otomatik ömürlü nesneye referans!
}
```

| Return İfadesi | `decltype` Formu | Sonuç | Güvenli? |
|---|---|---|---|
| `return x;` | identifier | `int` | ✓ |
| `return (x);` | expression (lvalue) | `int&` | ❌ UB — dangling reference |

> **Önemli Not:** `return` ifadesinde parantez kullanmak, `decltype(auto)` bağlamında **identifier kuralından expression kuralına geçiş** yaratır. `(x)` bir lvalue expression olduğundan `int&` çıkarılır ve otomatik ömürlü bir nesneye referans döndürülür — tanımsız davranış. Generic kodda `return` ifadelerini parantez içine alırken son derece dikkatli olunmalıdır.

### 2.3 prvalue İfadeleri — Güvenli

```cpp
decltype(auto) foo(int x) {
    return x + 1;   // ✓ prvalue → decltype(x+1) = int
}
```

`x + 1` bir prvalue olduğu için `decltype` `int` döndürür. Güvenlidir.

### 2.4 Unary `+` Operatörü — **Tanımsız Davranış!**

```cpp
decltype(auto) foo(int x) {
    return +x;   // ❌ +x lvalue → decltype(+x) = int& → dangling reference
}
```

Unary `+` operatörü lvalue üretir; `decltype` sonucu `int&` olur — UB.

### 2.5 Ternary Operatör — Value Category'ye Dikkat

```cpp
// ✓ Güvenli — her iki operand da prvalue ise ifade prvalue
decltype(auto) foo(int x) {
    return x >= 0 ? x : 0;  // prvalue → int
}

// ❌ Tehlikeli — her iki operand da lvalue ise ifade lvalue
decltype(auto) bar(int x, int y) {
    return x > y ? x : y;   // lvalue → int& → dangling reference!
}
```

| 2. ve 3. Operand | Ternary İfadenin Value Category'si | `decltype(auto)` Sonucu |
|---|---|---|
| İkisi de prvalue | prvalue | `T` ✓ |
| İkisi de lvalue | lvalue | `T&` ❌ (otomatik ömürlü ise) |
| Biri lvalue, biri prvalue | prvalue | `T` ✓ |

### 2.6 Member Access — xvalue Tuzağı

```cpp
struct Net {
    int x = 0;
};

// ✓ Güvenli — identifier (member access dahil)
decltype(auto) foo() {
    return Net{}.x;     // identifier kuralı → int
}

// ❌ Tehlikeli — parantez içi expression
decltype(auto) bar() {
    return (Net{}.x);   // xvalue → int&& → geçici nesneye rvalue ref → UB!
}
```

**Rvalue sınıf nesnesinin non-static data member'ına erişim ifadesi xvalue'dur.** Parantez içine alındığında `decltype` expression kuralı uygulanır:

| İfade | Value Category | `decltype` Sonucu | Güvenli? |
|---|---|---|---|
| `Net{}.x` | identifier (member access) | `int` | ✓ |
| `(Net{}.x)` | xvalue (expression) | `int&&` | ❌ UB |

> **Önemli Not:** `decltype(auto)` kullandığınızda assembly düzeyinde referans = adres döndürmek demektir. Geçici nesneye veya otomatik ömürlü nesneye adres döndürmek nasıl UB ise, referans döndürmek de aynı şekilde UB'dir. `int&` ile `int&&` arasında bu açıdan fark yoktur — her ikisi de ömrü biten nesneye bağlanırsa tanımsız davranış oluşturur.

---

## 3. Overload Resolution'da Universal Reference Önceliği

### 3.1 Temel Kural

Universal reference parametreli fonksiyon şablonu, overload resolution'da **her zaman ikinci öncelikte** yer alır — exact match olan non-template overload varsa o seçilir, yoksa universal reference kazanır:

```cpp
void foo(MyClass&)        { std::println("MyClass&"); }       // (1)
void foo(const MyClass&)  { std::println("const MyClass&"); } // (2)
void foo(MyClass&&)       { std::println("MyClass&&"); }      // (3)
void foo(const MyClass&&) { std::println("const MyClass&&"); }// (4)

template<typename T>
void foo(T&&)             { std::println("T&&"); }            // (5) Universal ref
```

### 3.2 Overload Resolution Tablosu

| Argüman | 1. Tercih | 2. Tercih (univ. ref yokken) | Universal Ref ile 2. Tercih |
|---|---|---|---|
| `MyClass` lvalue | (1) `MyClass&` | (2) `const MyClass&` | **(5) `T&&`** |
| `const MyClass` lvalue | (2) `const MyClass&` | — (sentaks hatası) | **(5) `T&&`** |
| `MyClass` rvalue | (3) `MyClass&&` | (4) `const MyClass&&` | **(5) `T&&`** |
| `const MyClass` rvalue | (4) `const MyClass&&` | (2) `const MyClass&` | **(5) `T&&`** |

Universal reference parametreli overload eklendiğinde, birinci tercih değişmez; ancak **ikinciliği her zaman universal reference alır**.

> **Önemli Not:** Bu davranış, niyetiniz dışında universal reference overload'un seçilmesine yol açabilir. Exact match olan overload'u kaldırdığınızda, `const T&` yerine `T&&` template'i devreye girer. Bu nedenle universal reference overload'ları mutlaka concepts veya SFINAE ile kısıtlanmalıdır.

---

## 4. `std::ref` ve `std::reference_wrapper`

### 4.1 Problem: STL Algoritmalarında Kopyalama Maliyeti

Birçok STL algoritması predicate/function object parametresini **değer semantiği** ile alır:

```cpp
template<typename InputIt, typename Pred>
auto count_if(InputIt first, InputIt last, Pred f) -> /* ... */;
// Pred f → DEĞER parametresi, kopyalama yapılır!
```

Büyük callable nesneler için bu ciddi performans kaybına yol açar:

```cpp
struct BigPred {
    char buffer[10'000];           // Kopyalama maliyeti yüksek
    bool operator()(int x) const { return x > 0; }
};

BigPred pred;
// ❌ pred kopyalanır — 10KB kopyalama maliyeti
auto n = std::ranges::count_if(vec, pred);
```

### 4.2 Çözüm 1: `std::ref`

```cpp
// ✓ std::reference_wrapper<BigPred> kopyalanır — sadece pointer boyutu
auto n = std::ranges::count_if(vec, std::ref(pred));
```

`std::ref(pred)` → `std::reference_wrapper<BigPred>` döndürür. Bu wrapper:
- Sadece bir pointer tutar (lightweight)
- `operator()` yönlendirmesi yapar
- Tür dönüştürme operatörü ile orijinal türe referans döndürür

### 4.3 Çözüm 2: Explicit Template Argümanı

```cpp
// ✓ Template argümanı referans türü yapılır — kopyalama yok
auto n = std::count_if<std::vector<int>::iterator, BigPred&>(
    vec.begin(), vec.end(), pred);
```

Bu yöntem çağrı görüntüsünü karmaşıklaştırır ama `std::ref` kullanmadan aynı etkiyi sağlar.

### 4.4 Taşıma (Move) Her Zaman Kazanç Sağlamaz

```cpp
struct BigPred {
    char buffer[10'000];  // C-style array — taşıma = kopyalama
    bool operator()(int x) const { return x > 0; }
};
```

| Veri Üyesi Türü | Move vs Copy Farkı |
|---|---|
| `std::string`, `std::vector` | Move **çok daha hızlı** (pointer swap) |
| `char[N]`, `int[N]` (C array) | Move = Copy (**fark yok**) |
| `std::array<T, N>` (POD elemanlar) | Move = Copy (**fark yok**) |
| `std::array<string, N>` | Move **daha hızlı** (elemanlar taşınır) |

> **Önemli Not:** Derleyicinin ürettiği move constructor, her üyeyi move eder. Ancak C-style array'ler ve `std::array<POD, N>` gibi türlerde move ile copy arasında **hiçbir performans farkı yoktur**. Move semantiğinden kazanç sağlamak için üyelerin kendilerinin move-optimizable olması gerekir.

---

## 5. Tag Dispatch — Empty Class ile Overload Seçimi

### 5.1 Problem

Aynı parametre türüne sahip birden fazla constructor overload'u gerektiğinde, fonksiyon overloading doğrudan kullanılamaz:

```cpp
class UniqueLock {
    std::mutex& mtx_;
public:
    // Hepsi std::mutex& parametreli — overload OLMAZ!
    UniqueLock(std::mutex& m);  // Edinir ve kilitler
    UniqueLock(std::mutex& m);  // Zaten kilitli, sadece edinir
    UniqueLock(std::mutex& m);  // Edinir ama kilitlemez
    UniqueLock(std::mutex& m);  // try_lock çağırır
};
```

### 5.2 Çözüm: Tag Class'lar

Boş sınıflar (tag class) ek parametre olarak eklenerek overload resolution sağlanır:

```cpp
// Tag type tanımları
struct defer_lock_t  {};    // Edinir, kilitlemez
struct adopt_lock_t  {};    // Zaten kilitli olanı edinir
struct try_to_lock_t {};    // try_lock ile edinmeyi dener

// constexpr tag nesneleri — compile-time sabitleri
inline constexpr defer_lock_t  defer_lock  {};
inline constexpr adopt_lock_t  adopt_lock  {};
inline constexpr try_to_lock_t try_to_lock {};
```

```cpp
class UniqueLock {
    std::mutex& mtx_;
public:
    // Farklı stratejiler — tag dispatch ile overload
    explicit UniqueLock(std::mutex& m)
        : mtx_(m) { mtx_.lock(); }

    UniqueLock(std::mutex& m, defer_lock_t)
        : mtx_(m) { /* kilitlemez */ }

    UniqueLock(std::mutex& m, adopt_lock_t)
        : mtx_(m) { /* zaten kilitli */ }

    UniqueLock(std::mutex& m, try_to_lock_t)
        : mtx_(m) { mtx_.try_lock(); }
};
```

### 5.3 Client Code Kullanımı

```cpp
std::mutex mtx;

// Strateji seçimi — tag nesnesiyle
UniqueLock lk1(mtx);                  // Doğrudan kilitle
UniqueLock lk2(mtx, defer_lock);      // Kilitlemeden edin
UniqueLock lk3(mtx, adopt_lock);      // Zaten kilitli olanı edin
UniqueLock lk4(mtx, try_to_lock);     // Kilitlemeyi dene
```

### 5.4 Tag Dispatch — Standart Kütüphane Örnekleri

| Sınıf/Fonksiyon | Tag Türü | Amaç |
|---|---|---|
| `std::unique_lock` | `defer_lock_t`, `adopt_lock_t`, `try_to_lock_t` | Mutex edinme stratejisi |
| `std::optional` | `std::in_place_t` | In-place construction |
| `std::variant` | `std::in_place_type_t<T>`, `std::in_place_index_t<I>` | Tür/indeks bazlı construction |
| `std::any` | `std::in_place_type_t<T>` | In-place construction |

### 5.5 Konvensiyon

| Öğe | Konvensiyon | Örnek |
|---|---|---|
| Tag type ismi | `*_t` suffix | `defer_lock_t` |
| Tag nesnesi ismi | Suffix'siz | `defer_lock` |
| Tag nesnesi niteliği | `inline constexpr` | `inline constexpr defer_lock_t defer_lock{};` |

> **Önemli Not:** Tag dispatch, zero-overhead bir tekniktir. Boş sınıfların boyutu 1 byte olsa da, `constexpr` nesneler compile-time'da çözülür ve optimizasyonla sıfır runtime maliyetine düşer. Standart kütüphane bu tekniği yaygın olarak kullanır; kendi API tasarımlarınızda da aynı konvensiyonu (`_t` suffix + `constexpr` nesne) uygulamanız önerilir.

---

## 6. User-Defined Literals (UDL)

### 6.1 Kavram

C++'ta built-in literal'ler (`42`, `3.14`, `'a'`, `"hello"`) derleyici tarafından temel türlere dönüştürülür. **User-defined literals**, bu mekanizmayı genişleterek programcı tanımlı türler için literal sentaksı sunar:

```cpp
#include <string>
using namespace std::string_literals;

auto name = "Ahmet"s;  // std::string türünde — "s" suffix'i
// Eşdeğer: auto name = std::string("Ahmet");
```

### 6.2 Literal Operator Function

UDL mekanizması arka planda bir **literal operator function** çağırır:

```cpp
// Derleyicinin dönüştürdüğü çağrı:
"Ahmet"s  →  operator""s("Ahmet", 5)
```

### 6.3 Literal Operator Function — Parametre Kuralları

| Literal Türü | İzin Verilen Parametre Formları |
|---|---|
| **Integer** | `unsigned long long` veya `const char*` (raw) |
| **Floating-point** | `long double` veya `const char*` (raw) |
| **Character** | `char`, `wchar_t`, `char8_t`, `char16_t`, `char32_t` |
| **String** | `const char*, std::size_t` |

### 6.4 Cooked vs Raw Form

```cpp
// Cooked form — derleyici literal'i sayısal değere dönüştürüp gönderir
constexpr Meter operator""_m(long double val) {
    return Meter{val};
}

// Raw form — derleyici literal'i string olarak gönderir
constexpr BigInt operator""_big(const char* str) {
    return BigInt::parse(str);  // "12345678901234567890" → BigInt
}
```

| Form | Parametre | Avantaj |
|---|---|---|
| **Cooked** | `unsigned long long` / `long double` | Basit, doğrudan değer |
| **Raw** | `const char*` | Precision kaybı yok, büyük sayılar desteklenir |

### 6.5 Suffix İsimlendirme Kuralları

```cpp
// ❌ Underscore ile başlamayan suffix — standart kütüphaneye ayrılmış
constexpr auto operator""s(const char* str, std::size_t len);  // STL'ye ait

// ✓ Underscore ile başlayan suffix — kullanıcı tanımlı
constexpr Meter operator""_m(long double val);
constexpr Kilogram operator""_kg(long double val);
```

| Suffix | Ait Olduğu Alan |
|---|---|
| Underscore ile **başlamayan** (`s`, `h`, `min`) | Standart kütüphane (reserved) |
| Underscore ile **başlayan** (`_m`, `_kg`, `_date`) | Kullanıcı tanımlı |

> **Önemli Not:** C++ standardı underscore ile başlamayan suffix'leri standart kütüphaneye ayırmıştır. Kullanıcı tanımlı literal'lerde **mutlaka** underscore prefix kullanılmalıdır (`_m`, `_km`, `_kg` vb.). Aksi halde isim çakışması ve tanımsız davranış riski oluşur.

### 6.6 Strong Typing ile UDL Kullanımı

```cpp
struct Meter {
    double value;
    constexpr explicit Meter(double v) : value(v) {}
};

struct Kilogram {
    double value;
    constexpr explicit Kilogram(double v) : value(v) {}
};

constexpr Meter operator""_m(long double v) {
    return Meter{static_cast<double>(v)};
}

constexpr Kilogram operator""_kg(long double v) {
    return Kilogram{static_cast<double>(v)};
}

// Derleme zamanı güvenliği
constexpr auto distance = 5.0_m;     // Meter türü
constexpr auto weight   = 72.5_kg;   // Kilogram türü

void apply_force(Meter m, Kilogram kg);
apply_force(5.0_m, 72.5_kg);    // ✓ Doğru türler
// apply_force(5.0_kg, 72.5_m); // ❌ Sentaks hatası — tür uyumsuzluğu
```

> **Önemli Not:** UDL'ler strong typing için güçlü bir araçtır. Fiziksel birimler (metre, kilogram, saniye), para birimleri, koordinat sistemleri gibi kavramsal olarak farklı ama aynı primitive türle temsil edilen değerler için **compile-time tür güvenliği** sağlar. Low-latency sistemlerde yanlış birim karıştırma hataları runtime'da tespit edilemez; UDL + strong typing bu hataları compile time'a taşır.

### 6.7 Numeric Literal Operator Template (C++14)

```cpp
// Template form — her karakter ayrı template argümanı olarak gelir
template<char... Chars>
constexpr auto operator""_b() {
    // Binary literal: 1010_b → {1, 0, 1, 0}
    return parse_binary<Chars...>();
}

auto val = 1010_b;  // operator""_b<'1','0','1','0'>()
```

### 6.8 String UDL ve C++20 Genişlemesi

C++20 ile string literal operator template'leri desteklenir:

```cpp
// C++20: String literal operator template
template<std::size_t N>
struct FixedString {
    char data[N];
    constexpr FixedString(const char (&str)[N]) {
        std::ranges::copy(str, data);
    }
};

template<FixedString S>
constexpr auto operator""_fs() {
    return S;
}

constexpr auto greeting = "hello"_fs;  // Compile-time string
```

### 6.9 Standart Kütüphane UDL'leri

```cpp
// <string> — string literals
using namespace std::string_literals;
auto s = "hello"s;                    // std::string

// <string_view> — string_view literals
using namespace std::string_view_literals;
auto sv = "hello"sv;                  // std::string_view

// <chrono> — duration literals
using namespace std::chrono_literals;
auto duration = 500ms;                // std::chrono::milliseconds
auto timeout  = 2s;                   // std::chrono::seconds
auto delay    = 100us;                // std::chrono::microseconds
auto period   = 1h + 30min;          // std::chrono::hours + minutes

// <complex> — complex literals (C++14)
using namespace std::complex_literals;
auto z = 3.0 + 4.0i;                 // std::complex<double>
```

| Namespace | Suffix'ler | Tür |
|---|---|---|
| `std::string_literals` | `s` | `std::string` |
| `std::string_view_literals` | `sv` | `std::string_view` |
| `std::chrono_literals` | `h`, `min`, `s`, `ms`, `us`, `ns` | `std::chrono::duration` |
| `std::complex_literals` | `i`, `il`, `if` | `std::complex` |
| `std::literals` | Tümü | Hepsini içerir |

> **Önemli Not:** `using namespace std::literals` tüm standart UDL'leri görünür kılar. Ancak global scope'ta `using namespace` kullanmak isim çakışması riski taşır. **Fonksiyon scope'unda** veya **blok scope'unda** kullanmak en güvenli yaklaşımdır. Header dosyalarında kesinlikle global scope'ta kullanılmamalıdır.

---

## 7. `constexpr` ve UDL — Compile-Time Değerlendirme

```cpp
constexpr Meter operator""_m(long double v) {
    return Meter{static_cast<double>(v)};
}

// Compile-time sabiti
constexpr auto track_length = 100.0_m;   // constexpr Meter
static_assert(track_length.value == 100.0);

// consteval ile zorunlu compile-time (C++20)
consteval Meter operator""_cm(long double v) {
    return Meter{static_cast<double>(v) / 100.0};
}

constexpr auto height = 175.0_cm;  // Zorunlu compile-time
```

> **Önemli Not:** `constexpr` literal operator fonksiyonları hem compile-time hem runtime'da çağrılabilir. `consteval` (C++20) ise **sadece compile-time** çağrıyı zorunlu kılar. Performans-kritik sistemlerde tüm birim dönüşümlerinin compile-time'da yapılmasını garanti etmek için `consteval` tercih edilmelidir.

---

## 8. Özet Tablosu

| Konu | Anahtar Kavram | Tehlike/Dikkat |
|---|---|---|
| Perfect returning | `decltype(auto)` return type | Parantez kullanımı UB yaratabilir |
| `decltype(auto)` + `(x)` | Expression kuralı devreye girer | `int&` → dangling reference |
| `decltype(auto)` + member access | rvalue nesne → xvalue | `int&&` → dangling reference |
| Ternary + `decltype(auto)` | İki operand lvalue → lvalue | `T&` → otomatik ömürlü nesneye ref |
| Universal ref overload | Her zaman 2. öncelik | İstenmeyen template instantiation |
| `std::ref` | `reference_wrapper` döndürür | Kopyalama maliyetini ortadan kaldırır |
| Tag dispatch | Boş sınıf + `constexpr` nesne | Zero-overhead overload seçimi |
| UDL suffix kuralı | `_` prefix zorunlu (kullanıcı) | Standart kütüphane ile çakışma riski |
| UDL + strong typing | Compile-time tür güvenliği | Birim karıştırma hataları derleme zamanında yakalanır |
