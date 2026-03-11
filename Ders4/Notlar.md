# Ders 4 — noexcept ve Performans, Mandatory Copy Elision, Temporary Materialization, NRVO Detayları

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 7 Nisan 2025
> **Kapsam:** noexcept'in reallocation performansına etkisi, strong exception guarantee, mandatory copy elision (C++17), temporary materialization, unmaterialized object passing, return value categories, NRVO koşulları, copy elision ve special member function ilişkisi

---

## 1. `noexcept` Move Constructor ve Reallocation Performansı

### 1.1 Problem: `std::vector` Reallocation Stratejisi

`std::vector::push_back` ve `std::vector::resize` gibi fonksiyonlar **strong exception guarantee** verir. Kapasite aşıldığında reallocation sırasında derleyici bir seçim yapar:

| Move Ctor `noexcept` mi? | Reallocation Stratejisi | Performans |
|---|---|---|
| **Evet** (`noexcept`) | Eski öğeler **taşınır** (move) | Hızlı |
| **Hayır** | Eski öğeler **kopyalanır** (copy) | Yavaş |

### 1.2 Neden Kopyalama Seçiliyor?

Strong exception guarantee = **commit-or-rollback**: işlem ya tamamen başarılı olur ya da nesne eski haline döner.

```
Reallocation sırasında taşıma senaryosu:

Eski bellek:  [A] [B] [C] [D]
                ↓   ↓   ↓
Yeni bellek:  [A'] [B'] [?]  ← 3. taşımada exception!

Problem: A ve B zaten taşındı (moved-from state).
         Geri taşıma da exception atabilir → rollback imkansız.
         Strong guarantee ihlal edilir.
```

Kopyalama ile: eski bellek **değişmez**, exception olursa yeni bellek serbest bırakılır → rollback güvenli.

### 1.3 Performans Farkı — Benchmark

```cpp
#include <vector>
#include <string>
#include <chrono>
#include <print>

class Wrapper {
    std::string ms;
public:
    Wrapper() : ms(500, 'A') {}
    Wrapper(const Wrapper&) = default;

    // noexcept OLMADAN: reallocation'da kopyalama yapılır
    // Wrapper(Wrapper&& other) : ms(std::move(other.ms)) {}

    // noexcept İLE: reallocation'da taşıma yapılır
    Wrapper(Wrapper&& other) noexcept : ms(std::move(other.ms)) {}
};

int main() {
    std::vector<Wrapper> vec;
    vec.resize(1'000'000);

    auto start = std::chrono::steady_clock::now();
    vec.resize(vec.capacity() + 1);  // reallocation zorla
    auto end = std::chrono::steady_clock::now();

    std::chrono::duration<double, std::milli> elapsed = end - start;
    std::println("Süre: {:.3f} ms", elapsed.count());
}
```

Tipik sonuçlar (1M eleman, 500-byte string):

| Durum | Süre |
|---|---|
| Move ctor `noexcept` **yok** | ~500 ms |
| Move ctor `noexcept` **var** | ~13 ms |

> **Önemli Not:** ~40x performans farkı! Bu, low-latency sistemlerde `noexcept` move constructor eksikliğinin kabul edilemez gecikme spike'larına yol açacağı anlamına gelir. **Her kaynak-tutan sınıfın move constructor'ı `noexcept` bildirilmelidir.**

---

## 2. Exception Guarantee Seviyeleri

| Garanti | Anlamı | Örnek |
|---|---|---|
| **No-throw** | Exception asla atılmaz | `noexcept` fonksiyonlar |
| **Strong (Commit-or-Rollback)** | İşlem başarısız olursa nesne eski haline döner | `vector::push_back` |
| **Basic** | Nesne geçerli (valid) durumda kalır, kaynak sızıntısı olmaz, ama eski değeri garanti edilmez | Çoğu STL operasyonu |
| **No guarantee** | Hiçbir garanti yok | Kaçınılmalı |

---

## 3. Derleyicinin `noexcept` Kararı (Implicit Declaration)

Derleyicinin implicitly declared ettiği special member function'lar (default ctor, copy ctor, move ctor, destructor vb.) `noexcept` olup olmadığı **member'ların `noexcept` durumuna** göre belirlenir:

```cpp
struct Inner {
    Inner() noexcept = default;  // noexcept ✓
};

struct Outer {
    Inner mx;
    // Derleyicinin yazdığı default ctor → mx'i default construct eder
    // Inner::Inner() noexcept olduğu için → Outer::Outer() de noexcept
};

// Doğrulama:
static_assert(noexcept(Outer{}));  // true

// Eğer Inner'ın default ctor'u noexcept DEĞİLSE:
struct InnerRisky {
    InnerRisky() {}  // noexcept değil (implicit)
};

struct OuterRisky {
    InnerRisky mx;
    // Derleyicinin default ctor'u da noexcept OLMAZ
};

static_assert(!noexcept(OuterRisky{}));  // true (noexcept değil)
```

### 3.1 `noexcept` Operatörü ile Compile-Time Sorgu

```cpp
#include <type_traits>

struct Widget {
    Widget() noexcept = default;
    Widget(const Widget&) = default;
    Widget(Widget&&) noexcept = default;
};

// noexcept operatörü: compile-time bool üretir
constexpr bool b1 = noexcept(Widget{});                    // true
constexpr bool b2 = noexcept(Widget(std::declval<Widget>())); // true (move)

// Destructor her zaman noexcept (user-declared bile olsa default)
static_assert(noexcept(std::declval<Widget>().~Widget()));
```

> **Önemli Not:** Destructor, explicit `noexcept` bildirimi olmasa bile **default olarak `noexcept`**'tir. Destructor'dan exception throw etmek neredeyse her zaman `std::terminate` çağrısına yol açar.

---

## 4. Hangi Fonksiyonlar `noexcept` Olmalı?

| Fonksiyon | `noexcept` Olmalı mı? | Neden |
|---|---|---|
| **Move constructor** | Evet | STL konteynerleri kopyalama yerine taşıma yapabilsin |
| **Move assignment** | Evet | Aynı sebep |
| **Swap fonksiyonları** | Evet | Genellikle pointer/değer takası — exception riski yok |
| **Destructor** | Otomatik evet | Standart default olarak `noexcept` yapar |
| **`operator delete`** | Evet | Exception sırasında bellek serbest bırakılırken çağrılır |

### 4.1 `operator delete` Neden `noexcept` Olmalı?

```cpp
struct Resource {
    Resource() { /* kaynak edinimi — exception atabilir */ }
    ~Resource() { /* kaynak serbest bırakma */ }
};

// new ifadesinde constructor exception atarsa:
// 1. Bellek operatör new ile tahsis edilmiş
// 2. Constructor exception attı → nesne oluşmadı → destructor çağrılmaz
// 3. Derleyici operatör delete'i çağırarak belleği serbest bırakır
// 4. Eğer operatör delete de exception atarsa → std::terminate!
```

### 4.2 `std::swap` İmplementasyonu

```cpp
template<typename T>
constexpr void swap(T& a, T& b)
    noexcept(std::is_nothrow_move_constructible_v<T> &&
             std::is_nothrow_move_assignable_v<T>)
{
    T tmp = std::move(a);
    a = std::move(b);
    b = std::move(tmp);
}
```

> **Önemli Not:** `std::swap` **conditionally noexcept**'tir — `T`'nin move constructor ve move assignment'ı `noexcept` ise `swap` de `noexcept` olur. Bu conditional pattern, kendi generic fonksiyonlarınızda da kullanılmalıdır.

---

## 5. Mandatory Copy Elision (C++17) — Temporary Materialization

### 5.1 Temel Kavram Değişikliği

C++17 ile **prvalue expression** artık doğrudan bir nesneyi temsil etmez. Prvalue, ancak belirli bağlamlarda **materialize** olarak (xvalue'ya dönüşerek) gerçek bir nesne haline gelir.

| Standart Öncesi (C++14) | C++17 Sonrası |
|---|---|
| prvalue = geçici nesne | prvalue = nesne **değil**, potansiyel değer |
| Copy elision = optimizasyon | Mandatory elision = **kural** (copy/move ctor gerekmez) |
| delete copy/move ctor → hata | delete copy/move ctor → **sorun değil** (prvalue bağlamında) |

### 5.2 Temporary Materialization Gereken Bağlamlar

Prvalue'nun xvalue'ya dönüşerek gerçek bir nesne haline geldiği durumlar:

| Bağlam | Örnek |
|---|---|
| Referansa bağlama | `const MyClass& r = MyClass{};` |
| Üye fonksiyon çağrısı | `MyClass{}.func();` |
| Üye erişimi | `MyClass{}.mx;` |
| Lambda init capture | `[x = MyClass{}]{}` |
| `typeid` operandı | `typeid(MyClass{})` |
| `sizeof` operandı | `sizeof(MyClass{})` |

### 5.3 Mandatory Elision Senaryoları

#### Senaryo 1: Unmaterialized Object Passing

Prvalue'nun fonksiyon parametresine doğrudan geçirilmesi — kopyalama/taşıma **yok**:

```cpp
#include <print>

class Heavy {
public:
    Heavy() { std::println("default ctor"); }
    Heavy(const Heavy&) = delete;  // Kopyalama yasak
    Heavy(Heavy&&) = delete;       // Taşıma yasak
};

void process(Heavy h) {  // Parametre by-value
    std::println("processing");
}

int main() {
    process(Heavy{});  // OK! (C++17) — sadece default ctor çağrılır
    // Copy/move ctor delete olmasına rağmen ÇALIŞIR
}
```

#### Senaryo 2: Prvalue Return (Return Value Optimization)

```cpp
Heavy create() {
    return Heavy{};  // prvalue döndürme → mandatory elision
}

int main() {
    Heavy h = create();  // Sadece 1 default ctor çağrılır
    // İç içe bile olsa:
    process(create());   // Yine sadece 1 default ctor
}
```

#### Senaryo 3: Zincirleme Prvalue

```cpp
Heavy create_chain() {
    return Heavy{Heavy{Heavy{}}};  // 3 iç içe prvalue
    // C++17: Sadece 1 default ctor çağrılır!
    // Tüm ara kopyalama/taşıma elide edilir (mandatory)
}
```

> **Önemli Not:** "Mandatory copy elision" terimi teknik olarak yanlıştır — C++17'de prvalue bağlamında **kopyalama hiç oluşmaz**, dolayısıyla "elide" edilecek bir şey yoktur. Standart buna **"simplified value categories"** veya **"guaranteed copy elision"** der. Ama sektörde yaygın terim mandatory copy elision olarak kalmıştır.

### 5.4 Mandatory Elision'ın Geçerli OLMADIĞI Durumlar

```cpp
class Widget {
public:
    Widget() = default;
    Widget(const Widget&) { std::println("copy"); }
    Widget(Widget&&) noexcept { std::println("move"); }
};

Widget create_named() {
    Widget w;        // isimlendirilmiş değişken (lvalue)
    return w;        // NRVO uygulanabilir AMA mandatory DEĞİL
}

Widget identity(Widget w) {
    return w;        // Parametre → mandatory elision yok, implicit move var
}

int main() {
    Widget a = create_named();  // NRVO veya move (derleyiciye bağlı)
    Widget b = identity(Widget{});  // Argüman: mandatory elision
                                     // Return: implicit move
}
```

---

## 6. NRVO (Named Return Value Optimization) — Detaylı Analiz

### 6.1 NRVO Koşulları

NRVO uygulanabilmesi için:

1. Döndürülen nesne **otomatik ömürlü yerel değişken** olmalı
2. Değişkenin türü, fonksiyonun geri dönüş türüyle **aynı** olmalı (cv-qualification farkı hariç)
3. Değişken fonksiyon **parametresi olmamalı**
4. Tek bir return path olması NRVO'yu kolaylaştırır (zorunlu değil)

### 6.2 NRVO Uygulanamayan Durumlar

```cpp
struct Widget {
    Widget() { std::println("default ctor"); }
    Widget(const Widget&) { std::println("copy ctor"); }
    Widget(Widget&&) noexcept { std::println("move ctor"); }
};

// Durum 1: Birden fazla return path — NRVO uygulanmayabilir
Widget case1(bool flag) {
    Widget a, b;
    return flag ? a : b;  // Hangisi? Derleyici karar veremeyebilir
    // Sonuç: move ctor çağrılır (implicit move)
}

// Durum 2: Pessimistic move — NRVO ENGELLENİR
Widget case2() {
    Widget w;
    return std::move(w);  // YANLIŞ! NRVO yok, move ctor çağrılır
}

// Durum 3: Parametre döndürme — NRVO yok
Widget case3(Widget w) {
    return w;  // NRVO yok, ama implicit move var (C++11+)
}

// Durum 4: Tür uyumsuzluğu
struct Derived : Widget {};
Widget case4() {
    Derived d;
    return d;  // NRVO yok (farklı tür), slicing + copy/move
}
```

### 6.3 Copy Elision Kategorileri Özet

| Kategori | C++17 Öncesi | C++17 Sonrası | copy/move ctor gerekir mi? |
|---|---|---|---|
| **Prvalue → değişken** (`T x = T{}`) | Optimizasyon (isteğe bağlı) | **Mandatory** | Hayır |
| **Prvalue → parametre** (`f(T{})`) | Optimizasyon (isteğe bağlı) | **Mandatory** | Hayır |
| **Prvalue return** (`return T{}`) | Optimizasyon (isteğe bağlı) | **Mandatory** | Hayır |
| **NRVO** (`return named_var`) | Optimizasyon (isteğe bağlı) | Optimizasyon (isteğe bağlı) | Evet (fallback) |
| **Parametre return** (`return param`) | Copy | Implicit move | Evet |

> **Önemli Not:** As-if rule, derleyicinin observable behavior'u korumak koşuluyla optimizasyon yapmasına izin verir. Copy elision bu kuralın **istisnasıdır** — copy/move constructor'da side effect olsa bile (örneğin `std::println` çağrısı), derleyici bu constructor'ı atlayabilir. Bu, C++ standardında açıkça izin verilen nadir durumlardan biridir.

---

## 7. Liskov Substitution Principle ve `noexcept` (SOLID - L)

Derste gelen soruya cevaben kısa bir not:

- **LSP (Liskov Substitution Principle)**: Taban sınıf nesnesinin kullanıldığı her yerde türemiş sınıf nesnesi kullanılabilmeli.
- **Promise no more, require no less**: Türemiş sınıf, taban sınıfın vaatlerinden daha az vaat etmemeli; daha fazla ön koşul talep etmemeli.
- Bu ilke **is-a relationship** (kalıtım) için geçerlidir. **has-a relationship** (containment/composition) için geçerli değildir.

```cpp
// LSP ile noexcept ilişkisi:
struct Base {
    virtual void process() noexcept = 0;  // "exception atmam" vaadi
    virtual ~Base() = default;
};

struct Derived : Base {
    // Taban noexcept vaat etti → türemiş de noexcept olmalı
    void process() noexcept override { /* ... */ }

    // Eğer noexcept olmazsa: LSP ihlali
    // void process() override { /* exception atabilir! */ }  // Kötü tasarım
};
```

---

## 8. `std::chrono` ile Süre Ölçümü (Performans Profiling)

```cpp
#include <chrono>
#include <thread>
#include <print>

int main() {
    using namespace std::chrono_literals;

    // Ölçüm öncesi warm-up (opsiyonel, cache/scheduler etkisini azaltır)
    std::this_thread::sleep_for(1000ms);

    auto start = std::chrono::steady_clock::now();

    // ... ölçülecek işlem ...

    auto end = std::chrono::steady_clock::now();

    // Millisaniye cinsinden (double precision)
    std::chrono::duration<double, std::milli> elapsed = end - start;
    std::println("Süre: {:.3f} ms", elapsed.count());

    // C++20: doğrudan stream'e yazdırma (inserter desteği)
    std::println("Süre: {}", end - start);
}
```

### 8.1 Neden `steady_clock`?

| Clock | Özellik | Kullanım |
|---|---|---|
| `system_clock` | Sistem saati, ayarlanabilir | Tarih/saat gösterimi |
| `steady_clock` | **Monoton**, ayarlanamaz | **Süre ölçümü** |
| `high_resolution_clock` | En yüksek çözünürlük (genellikle `steady_clock` alias'ı) | Süre ölçümü |

> **Önemli Not:** `steady_clock` kullanılmalıdır çünkü `system_clock` NTP senkronizasyonu veya kullanıcı müdahalesiyle geri/ileri alınabilir. Bu, ölçülen sürenin negatif çıkmasına veya yanlış olmasına yol açar.

---

## 9. `std::chrono` Duration ve Ratio Sistemi

```cpp
#include <chrono>
#include <ratio>

// Duration = representation (sayı türü) + period (birim)
// std::chrono::duration<Rep, Period>

// Standart duration type alias'ları:
// std::chrono::nanoseconds  = duration<...int..., std::nano>    (10⁻⁹ s)
// std::chrono::microseconds = duration<...int..., std::micro>   (10⁻⁶ s)
// std::chrono::milliseconds = duration<...int..., std::milli>   (10⁻³ s)
// std::chrono::seconds      = duration<...int..., std::ratio<1>>
// std::chrono::minutes      = duration<...int..., std::ratio<60>>

// Özel duration türü:
using float_ms = std::chrono::duration<double, std::milli>;
// representation: double (kesirli kısım kaybolmaz)
// period: milli = ratio<1, 1000> = milisaniye

// Dönüşüm kuralı:
// İnce → kaba: örtülü dönüşüm YOK (bilgi kaybı riski)
// Kaba → ince: örtülü dönüşüm VAR
// double representation: her yönde örtülü dönüşüm VAR
```

---

## 10. Low-Latency Sistemler İçin Kritik Noktalar

> **Önemli Not:** `noexcept` move constructor'ın yokluğu, `std::vector` reallocation'larında **40-50x yavaşlamaya** neden olabilir (benchmark'ta 13ms vs 500ms). HFT, oyun motorları ve embedded real-time sistemlerde bu fark kabul edilemez. **Rule of thumb: Her kaynak-tutan sınıfın move constructor'ı ve move assignment'ı `noexcept` bildirilmelidir.**

> **Önemli Not:** Mandatory copy elision (C++17) sayesinde, prvalue sınıf nesneleri **sıfır kopyalama maliyetiyle** fonksiyonlara geçirilebilir ve fonksiyonlardan döndürülebilir. Bu, factory pattern'larda ve builder pattern'larda performans kaygısı olmadan by-value semantik kullanmayı mümkün kılar.

> **Önemli Not:** NRVO, derleyicinin nesneyi doğrudan çağıranın stack frame'inde oluşturmasını sağlar. Ancak **garanti değildir**. Performans-kritik kodda NRVO'nun uygulanıp uygulanmadığını doğrulamak için `-fno-elide-constructors` derleyici flag'i ile test yapılmalı ve constructor çağrı sayıları izlenmelidir.

> **Önemli Not:** `noexcept` ihlali (runtime'da exception throw) **her zaman `std::terminate` çağrısıyla** sonuçlanır — exception yakalanmaz, stack unwinding yapılmaz. Bu nedenle `noexcept` bildirimi ancak **gerçekten exception atılmayacağı garanti edildiğinde** yapılmalıdır. Yanlış `noexcept` bildirimi, debug edilmesi çok zor crash'lere yol açar.
