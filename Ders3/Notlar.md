# Ders 3 — Reference Qualifiers, Move-Only Types, Move Iterator, Copy Elision (NRVO/RVO)

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 2 Nisan 2025
> **Kapsam:** Ref-qualified member functions, getter overload stratejisi, move-only types, `std::move_iterator`, `std::make_move_iterator`, copy elision (mandatory/NRVO), pessimistic move, implicit move kuralı

---

## 1. Reference Qualifiers (Ref-Qualified Member Functions)

### 1.1 Temel Kavram

Sınıfın non-static üye fonksiyonlarına `&` veya `&&` eklenerek, fonksiyonun **yalnızca belirli value category'deki** sınıf nesneleri için çağrılması sağlanır.

```cpp
class Widget {
public:
    void process() &;        // Sadece lvalue nesneler için çağrılabilir
    void process() const &;  // Sadece const lvalue nesneler için çağrılabilir
    void process() &&;       // Sadece rvalue nesneler için çağrılabilir
    void process() const &&; // Sadece const rvalue nesneler için (pratikte nadir)
};
```

### 1.2 Overload Resolution ile Kullanım

```cpp
#include <print>

class Nage {
public:
    void func() &       { std::println("lvalue ref qualified"); }
    void func() const & { std::println("const lvalue ref qualified"); }
    void func() &&      { std::println("rvalue ref qualified"); }
    void func() const &&{ std::println("const rvalue ref qualified"); }
};

int main() {
    Nage obj;
    const Nage cobj;

    obj.func();                      // lvalue ref qualified
    cobj.func();                     // const lvalue ref qualified
    Nage{}.func();                   // rvalue ref qualified
    std::move(cobj).func();          // const rvalue ref qualified
}
```

### 1.3 Kritik Kural: Ref-Qualified ve Non-Qualified Birlikte Olamaz

```cpp
class Bad {
    void func();       // ref-qualifier yok
    void func() &;     // ref-qualifier var → SENTAKS HATASI!
    // "cannot overload a member function with ref-qualifier
    //  with a member function without ref-qualifier"
};
```

> **Önemli Not:** Bir fonksiyon adı için ref-qualified overload varsa, aynı isimde ref-qualifier'sız overload **bulunamaz**. Bu kural, overload resolution'da belirsizlik oluşmasını engeller.

### 1.4 Pratikte En Sık Kullanılan Overload Çifti

Dört overload'un hepsi nadiren gerekir. En yaygın kullanım **lvalue vs rvalue** ayrımıdır:

```cpp
class Sentence {
    std::string ms;
public:
    // lvalue nesneler için: referans döndür (kopyalama yok)
    const std::string& str() const & { return ms; }

    // rvalue nesneler için: taşıyarak döndür (kaynak çalınır)
    std::string str() && { return std::move(ms); }
};
```

---

## 2. Getter Fonksiyonları ve Dangling Reference Problemi

### 2.1 Getter'ın Geri Dönüş Türü Seçimi

| Geri Dönüş Türü | Maliyet | Risk |
|---|---|---|
| `T` (by value) | Deep copy | Güvenli |
| `const T&` (const lvalue ref) | Pointer kopyalama (~8 byte) | **Dangling reference riski** |
| `T` for rvalue / `const T&` for lvalue | Optimal | Güvenli (ref-qualified overload) |

### 2.2 Dangling Reference Senaryosu — Range-Based For Loop

```cpp
#include <string>

class Sentence {
    std::string ms;
public:
    Sentence(const char* p) : ms{p} {}
    const std::string& str() const { return ms; }
};

Sentence get_sentence() {
    return Sentence{"Merhaba Dunya"};
}

int main() {
    // TEHLİKE: Dangling reference!
    for (char c : get_sentence().str()) {
        // get_sentence() → geçici Sentence nesnesi
        // .str() → geçici nesnenin ms'ine const ref döndürür
        // Geçici nesne yok edilir → ref artık geçersiz!
    }
}
```

**Derleyicinin range-based for loop için ürettiği kod:**

```cpp
{
    auto&& range = get_sentence().str();  // forwarding reference
    auto iter = range.begin();
    auto end  = range.end();
    for (; iter != end; ++iter) {
        char c = *iter;
        // ...
    }
}
// get_sentence()'dan dönen geçici nesne bu noktada zaten yok edilmiş!
// range, dangling reference'a dönüşmüş durumda
```

> **Önemli Not:** `auto&&` forwarding reference'tır, `const auto&&` **değildir**. `auto&&` kullanılmasının nedeni hem lvalue hem rvalue ifadeleri kabul edebilmesidir. Ancak burada fonksiyonun `const T&` döndürmesi nedeniyle **lifetime extension uygulanmaz** — referans yoluyla döndürülen geçici nesnenin ömrü uzatılmaz.

### 2.3 Çözüm: Ref-Qualified Getter Overloads

```cpp
class Sentence {
    std::string ms;
public:
    Sentence(const char* p) : ms{p} {}

    // lvalue nesneler: referans döndür (kopyalama yok, verimli)
    const std::string& str() const & { return ms; }

    // rvalue nesneler: değer olarak döndür, kaynağı taşı
    std::string str() && { return std::move(ms); }
};
```

Artık `get_sentence().str()` çağrısında:
- `get_sentence()` → prvalue → rvalue nesne
- `str() &&` overload'u seçilir → `std::move(ms)` ile **taşıma** yapılır
- Geri dönüş değeri `std::string` (by value) → lifetime extension uygulanır

> **Önemli Not:** Bu pattern, `std::optional::value()`, `std::expected::value()` gibi standart kütüphane sınıflarında da kullanılır. Rvalue nesnenin member'ını taşımak güvenlidir çünkü nesne zaten ölmek üzeredir.

---

## 3. Üye Erişimi ve Value Category — Detay Kurallar

### 3.1 Non-Static Data Member Erişimi

| Nesne | Üye Türü | Erişim İfadesinin Value Cat. |
|---|---|---|
| lvalue | non-static, non-ref | **lvalue** |
| rvalue | non-static, non-ref | **xvalue** (→ rvalue) |
| lvalue/rvalue | **static** | **lvalue** (her zaman) |
| rvalue | non-static **referans** | **lvalue** (referans üye) |

```cpp
#include <utility>

struct Inner { int val; };

struct Outer {
    Inner mx;
    static inline Inner smx{};
};

int main() {
    Outer obj;

    // lvalue nesne → lvalue erişim
    auto a = obj.mx;                    // copy ctor

    // rvalue nesne → xvalue erişim
    auto b = Outer{}.mx;               // move ctor

    // std::move ile rvalue yapma → xvalue erişim
    auto c = std::move(obj).mx;        // move ctor

    // static member → HER ZAMAN lvalue
    auto d = Outer{}.smx;              // copy ctor (rvalue nesne olsa bile!)
    auto e = std::move(obj).smx;       // copy ctor (static = lvalue)
}
```

### 3.2 Taşıma İçin İki Eşdeğer Yazım

```cpp
Outer obj;
// Aşağıdaki ikisi eşdeğerdir:
auto x = std::move(obj).mx;       // rvalue nesnenin üyesine erişim → xvalue
auto y = std::move(obj.mx);       // üyeyi doğrudan rvalue'ya cast et → xvalue
```

> **Önemli Not:** Static data member'lar sınıf nesnesine değil sınıfa aittir; bir nesne yok edilse bile static member yaşamaya devam eder. Bu nedenle rvalue nesne üzerinden erişilse bile static member **her zaman lvalue**'dur — taşınması anlamsız ve tehlikelidir.

---

## 4. Move-Only Types

### 4.1 Tanım

Kopyalanamayan ama taşınabilen türler. Copy constructor ve copy assignment **deleted**, move constructor ve move assignment **tanımlı**.

### 4.2 Standart Kütüphanedeki Move-Only Türler

| Tür | Başlık Dosyası | Alan |
|---|---|---|
| `std::unique_ptr<T>` | `<memory>` | Bellek yönetimi |
| `std::jthread` | `<thread>` | Concurrency (C++20) |
| `std::thread` | `<thread>` | Concurrency |
| `std::unique_lock<M>` | `<mutex>` | Concurrency |
| `std::shared_lock<M>` | `<shared_mutex>` | Concurrency |
| `std::packaged_task<F>` | `<future>` | Concurrency |
| `std::future<T>` | `<future>` | Concurrency |
| `std::promise<T>` | `<future>` | Concurrency |

Not: `std::mutex` hem kopyalanamaz hem taşınamaz — move-only **değildir**.

### 4.3 Move-Only Sınıf Bildirimi

```cpp
// Yöntem 1: Sadece move member'ları bildir (copy member'lar otomatik delete edilir)
class MoveOnly {
public:
    MoveOnly() = default;
    MoveOnly(MoveOnly&&) noexcept = default;
    MoveOnly& operator=(MoveOnly&&) noexcept = default;
};

// Yöntem 2: Explicit delete + explicit default (daha okunabilir)
class MoveOnlyExplicit {
public:
    MoveOnlyExplicit() = default;
    MoveOnlyExplicit(const MoveOnlyExplicit&) = delete;
    MoveOnlyExplicit& operator=(const MoveOnlyExplicit&) = delete;
    MoveOnlyExplicit(MoveOnlyExplicit&&) noexcept = default;
    MoveOnlyExplicit& operator=(MoveOnlyExplicit&&) noexcept = default;
};
```

### 4.4 Move-Only Türlerin Kısıtlamaları

| Senaryo | Geçerli mi? | Neden |
|---|---|---|
| `MoveOnly y = x;` (x lvalue) | **HATA** | Kopyalama gerekir |
| `MoveOnly y = std::move(x);` | OK | Taşıma |
| `for (auto val : vec)` | **HATA** | Eleman kopyalanır |
| `for (auto& val : vec)` | OK | Referans, kopyalama yok |
| `for (auto&& val : vec)` | OK | Forwarding ref, kopyalama yok |
| `vec.push_back(lvalue)` | **HATA** | Copy overload seçilir |
| `vec.push_back(std::move(lvalue))` | OK | Move overload seçilir |
| `vec.push_back(MakeFunc())` | OK | Prvalue → rvalue |
| `vec.emplace_back(ctor_args...)` | OK | In-place construction |
| `std::initializer_list` ile init | **HATA** | `initializer_list` elemanları kopyalar |

```cpp
#include <vector>
#include <memory>
#include <string>

using uptr = std::unique_ptr<std::string>;

int main() {
    std::vector<uptr> v;

    uptr p = std::make_unique<std::string>("hello");

    // v.push_back(p);                  // HATA: lvalue → copy gerekir
    v.push_back(std::move(p));          // OK: rvalue → move
    v.push_back(std::make_unique<std::string>("world")); // OK: prvalue
    v.emplace_back(new std::string{"foo"});              // OK: in-place
    v.emplace_back();                   // OK: default ctor ile oluştur (nullptr)

    // Dolaşma:
    for (const auto& ptr : v) {
        if (ptr)
            std::println("{}", *ptr);
        else
            std::println("(null)");
    }
}
```

---

## 5. `std::move_iterator` ve `std::make_move_iterator`

### 5.1 Adaptör Kavramı

| Adaptör Kategorisi | Örnekler |
|---|---|
| Iterator Adaptors | `reverse_iterator`, `const_iterator`, `move_iterator`, `back_insert_iterator` |
| Container Adaptors | `stack`, `queue`, `priority_queue`, `flat_map` (C++23) |
| Function Adaptors | `std::bind`, `std::not_fn`, `std::mem_fn` |

### 5.2 `move_iterator` — Dereference = rvalue

Normal iteratör dereference edildiğinde **lvalue** döner. `move_iterator` dereference edildiğinde **rvalue** (xvalue) döner.

```cpp
#include <iterator>
#include <vector>
#include <print>

struct Widget {
    Widget() = default;
    Widget(const Widget&) { std::println("copy ctor"); }
    Widget(Widget&&) noexcept { std::println("move ctor"); }
    Widget& operator=(const Widget&) { std::println("copy assign"); return *this; }
    Widget& operator=(Widget&&) noexcept { std::println("move assign"); return *this; }
};

int main() {
    std::vector<Widget> src(3);

    // Normal iteratör → dereference = lvalue → copy
    auto normal_it = src.begin();
    Widget a = *normal_it;              // copy ctor

    // Move iteratör → dereference = rvalue → move
    auto move_it = std::make_move_iterator(src.begin());
    Widget b = *move_it;                // move ctor
}
```

### 5.3 Oluşturma Yolları

```cpp
std::vector<Widget> vec(5);

// Yol 1: Explicit tür belirtme (zahmetli)
std::move_iterator<std::vector<Widget>::iterator> it1(vec.begin());

// Yol 2: decltype ile (orta zahmet)
std::move_iterator<decltype(vec.begin())> it2(vec.begin());

// Yol 3: Fabrika fonksiyonu (en temiz) ✓
auto it3 = std::make_move_iterator(vec.begin());

// Yol 4: CTAD (C++17+) ✓
std::move_iterator it4(vec.begin());
```

### 5.4 Algoritmalarda Kullanım

```cpp
#include <algorithm>
#include <vector>
#include <list>
#include <iterator>

int main() {
    std::vector<Widget> src(5);
    std::list<Widget> dst;

    // copy algoritması + move_iterator = move algoritması gibi davranır
    std::copy(
        std::make_move_iterator(src.begin()),
        std::make_move_iterator(src.end()),
        std::back_inserter(dst)
    );
    // Her eleman copy değil MOVE ile aktarılır

    // Konteyner constructor'ı ile toplu taşıma:
    std::vector<Widget> dst2(
        std::make_move_iterator(src.begin()),
        std::make_move_iterator(src.end())
    );
}
```

> **Önemli Not:** `std::move_iterator` kullanıldıktan sonra kaynak konteyner'deki nesneler **moved-from state**'tedir. Bu nesnelere erişmek tanımsız davranış **değildir** (atama yapılabilir, destructor çağrılır), ancak değerleri **belirsiz**dir. Performans-kritik kodda gereksiz moved-from nesneleri tutmamak için `clear()` veya `shrink_to_fit()` çağrılmalıdır.

---

## 6. Copy Elision (Kopyalama Eliminasyonu)

### 6.1 İki Kategori

| Kategori | Zorunluluk | Standart |
|---|---|---|
| **Mandatory Copy Elision** | Derleyici **yapmalı** | C++17+ |
| **Named RVO (NRVO)** | Derleyici **yapabilir** (isteğe bağlı) | C++11+ |

### 6.2 Mandatory Copy Elision (Zorunlu — C++17)

Prvalue ifadeler doğrudan hedef nesnenin bellek alanında **materialized** olur. Copy/move constructor'a hiç gerek yoktur.

```cpp
class Heavy {
public:
    Heavy() { std::println("default ctor"); }
    Heavy(const Heavy&) = delete;   // Kopyalama yasak!
    Heavy(Heavy&&) = delete;        // Taşıma yasak!
};

Heavy create() {
    return Heavy{};  // OK! (C++17) — prvalue, mandatory elision
}

int main() {
    Heavy h = create();  // Sadece 1 default ctor çağrılır
    // copy/move ctor delete olmasına rağmen ÇALIŞIR
}
```

**Mandatory elision geçerli olduğu durumlar:**
- Fonksiyon `return prvalue;` (isimsiz geçici nesne döndürme)
- `T obj = prvalue;` (prvalue ile başlatma)

### 6.3 Named Return Value Optimization (NRVO) — İsteğe Bağlı

İsimlendirilmiş yerel değişken döndürülürken, derleyici kopyalama/taşımayı **atlayabilir** (ama garanti değil).

```cpp
#include <print>

class Widget {
public:
    Widget() { std::println("default ctor"); }
    Widget(const Widget&) { std::println("copy ctor"); }
    Widget(Widget&&) noexcept { std::println("move ctor"); }
};

Widget create() {
    Widget w;       // isimlendirilmiş yerel değişken
    return w;       // NRVO uygulanabilir → doğrudan hedefte oluşturulur
}

int main() {
    Widget obj = create();
    // Tipik çıktı (NRVO uygulandığında): sadece "default ctor"
    // NRVO uygulanmazsa: "default ctor" + "move ctor"
}
```

### 6.4 NRVO'nun Uygulanamadığı Durumlar

```cpp
// Durum 1: Birden fazla return path, farklı nesneler
Widget create(bool flag) {
    Widget a, b;
    if (flag) return a;  // Hangisi? Derleyici NRVO uygulayamayabilir
    return b;
}

// Durum 2: return std::move(local) — PESİMİSTİK MOVE!
Widget create_bad() {
    Widget w;
    return std::move(w);  // YANLIŞ! NRVO'yu engeller, move ctor çağrılır
}

// Durum 3: Parametre döndürme — NRVO yok (ama implicit move var)
Widget pass_through(Widget w) {
    return w;  // NRVO yok, ama C++11+ implicit move uygulanır
}
```

> **Önemli Not:** `return std::move(local_var);` yazmak **pessimistic move** olarak adlandırılır ve bir anti-pattern'dir. NRVO'yu engelleyerek gereksiz bir move constructor çağrısına yol açar. Doğru yazım: `return local_var;` — derleyici NRVO uygulayamazsa bile otomatik olarak move eder (implicit move kuralı).

### 6.5 Implicit Move (Örtülü Taşıma) Kuralı

C++11'den itibaren: `return` ifadesinde döndürülen nesne **otomatik ömürlü yerel değişken** veya **fonksiyon parametresi** ise ve NRVO uygulanmazsa, derleyici önce **move constructor**'ı dener, yoksa copy constructor'ı kullanır.

```cpp
// Move-only type bile return edilebilir (NRVO veya implicit move ile)
std::unique_ptr<int> make_ptr() {
    auto p = std::make_unique<int>(42);
    return p;  // OK: implicit move (p yerel değişken)
    // return std::move(p); yazılabilir ama gereksiz — derleyici zaten move eder
}
```

### 6.6 C++23 Implicit Move Genişlemesi

C++23 ile implicit move kuralı genişletildi: artık **sadece `return`** değil, **`throw`** ifadelerinde de ve daha geniş bağlamlarda implicit move uygulanır.

```cpp
// C++23: throw ifadesinde de implicit move
void might_throw() {
    std::string err = "something failed";
    throw err;  // C++23: implicit move, kopyalama yerine taşıma
}
```

---

## 7. Pessimistic Move — Tanım ve Kaçınma

### 7.1 Nedir?

`return std::move(x);` ifadesinde `x` bir **yerel değişken** ise, `std::move` kullanmak NRVO optimizasyonunu engeller ve gereksiz bir move constructor çağrısına neden olur.

### 7.2 Pessimistic Move vs Doğru Kullanım

| Durum | `std::move` Gerekli mi? | Neden |
|---|---|---|
| `return local_var;` | **HAYIR** | NRVO veya implicit move zaten uygulanır |
| `return param;` | **HAYIR** | Implicit move uygulanır (C++11+) |
| `return data_member;` | **EVET** | Veri elemanı yerel değişken değil, implicit move yok |
| `return std::move(member);` in `&&` qualified func | **EVET** | Rvalue nesnenin üyesini taşımak gerekir |

```cpp
class Container {
    std::vector<int> data;
public:
    // DOĞRU: veri elemanını taşı (rvalue nesne için)
    std::vector<int> get_data() && { return std::move(data); }

    // DOĞRU: veri elemanının referansını döndür (lvalue nesne için)
    const std::vector<int>& get_data() const & { return data; }
};

std::vector<int> make_vector() {
    std::vector<int> v{1, 2, 3};
    return v;  // DOĞRU: NRVO veya implicit move
    // return std::move(v);  // YANLIŞ: pessimistic move
}
```

> **Önemli Not:** Derleyiciler (GCC, Clang, MSVC) `return std::move(local);` yazıldığında **uyarı** verir (`-Wpessimizing-move`). Low-latency sistemlerde bu uyarıyı `-Werror` ile hataya çevirmek, ekipteki herkesi doğru pattern'a zorlar.

---

## 8. Return Value Optimization Özet Tablosu

| Return İfadesi | Copy Elision | Move | Copy | Notlar |
|---|---|---|---|---|
| `return T{};` | **Mandatory** | — | — | C++17 garantili |
| `return local_var;` | NRVO (muhtemel) | Fallback | Son çare | Derleyiciye bağlı |
| `return std::move(local);` | **Engellenir!** | Zorlanır | — | Anti-pattern |
| `return param;` | Yok | Implicit move | Fallback | C++11+ |
| `return member;` | Yok | Yok | Uygulanır | `std::move` gerekir |
| `return std::move(member);` | Yok | Uygulanır | — | Doğru kullanım |

---

## 9. Low-Latency Sistemler İçin Kritik Noktalar

> **Önemli Not:** Range-based for loop'ta `auto` (by value) kullanmak, konteyner elemanlarının **kopyalanmasına** neden olur. Büyük nesneler (`std::string`, `std::vector`) için bu ciddi bir performans kaybıdır. Her zaman `const auto&` veya `auto&&` tercih edin. Move-only türlerde ise `auto` ile dolaşma **derleme hatasıdır**.

> **Önemli Not:** `std::initializer_list` elemanları `const` dizide saklanır ve **kopyalama gerektirir**. Move-only türler `initializer_list` ile kullanılamaz. Performans-kritik kodda büyük nesneler için `initializer_list` yerine `reserve()` + `emplace_back()` çağrıları tercih edilmelidir.

> **Önemli Not:** `std::move_iterator` ile bir range'i toplu taşıdıktan sonra kaynak konteyner hâlâ **eski boyutundadır** — elemanlar moved-from state'tedir ama bellek serbest bırakılmamıştır. Bellek baskısı yüksek ortamlarda `src.clear(); src.shrink_to_fit();` ile kaynağı derhal serbest bırakmak gerekir.

> **Önemli Not:** NRVO, derleyicinin nesneyi doğrudan çağıranın stack frame'inde oluşturmasını sağlar — bu da gereksiz move constructor çağrısını bile ortadan kaldırır. Tek bir `return` path'i olan fonksiyonlarda NRVO neredeyse **garanti**dir (standart tarafından zorunlu olmasa da). Birden fazla `return` path'i olan fonksiyonlarda NRVO uygulanmayabilir.
