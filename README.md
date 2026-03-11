# İleri C++ Kursu — Ders Notları (C++26 Uyumlu)

> Bu repo, İleri C++ kursunun ders notlarını, kod örneklerini ve alıştırmalarını içerir.
> Tüm kodlar C++26 standartlarına uygun olarak yazılmıştır.

---

## Ders İndeksi

| # | Konu | Klasör | Kapsam |
|---|---|---|---|
| 1 | [Value Categories, Referans Türleri ve `decltype`](./Ders1) | `Ders1/` | lvalue, prvalue, xvalue, glvalue, rvalue, referans bağlama kuralları, `decltype` tür çıkarımı, move semantiği giriş |
| 2 | [Move Semantics, `std::move` İmplementasyonu ve Special Member Functions](./Ders2) | `Ders2/` | Move semantiği motivasyonu, `std::move` iç yapısı, `remove_reference` meta function, alias template, reference collapsing, universal reference deduction, `noexcept`, special member functions |
| 3 | [Reference Qualifiers, Move-Only Types, Move Iterator, Copy Elision](./Ders3) | `Ders3/` | Ref-qualified member functions, getter overload stratejisi, dangling reference, move-only types, `std::move_iterator`, `std::make_move_iterator`, mandatory copy elision, NRVO, pessimistic move, implicit move kuralı |
| 4 | [noexcept ve Performans, Mandatory Copy Elision, Temporary Materialization](./Ders4) | `Ders4/` | noexcept reallocation etkisi, strong exception guarantee, mandatory copy elision (C++17), temporary materialization, unmaterialized object passing, NRVO koşulları, `std::chrono` süre ölçümü, LSP ve noexcept |
| 5 | [Perfect Forwarding, `std::forward` İmplementasyonu, Universal Reference](./Ders5) | `Ders5/` | Perfect forwarding kavramı, `std::forward` iç yapısı (iki overload), `std::move` vs `std::forward`, universal reference olmayan durumlar, explicit specialization ve overload resolution, `auto&&`, variadic perfect forwarding, concepts ile kısıtlama |
| 6 | [`std::forward` Detayları, Abbreviated Template Syntax, SFINAE ve Concepts](./Ders6) | `Ders6/` | `std::forward` `static_assert` güvenliği, abbreviated template syntax (C++20), `decltype` ile perfect forwarding, lambda'larda perfect forwarding, ref-qualified getter + forwarding, universal reference kısıtlama (`requires`, `enable_if`), belirli tür için universal reference (`is_same`, `is_convertible`, `remove_cvref`) |
| 7 | [Perfect Returning, `decltype(auto)` Tuzakları, Tag Dispatch ve User-Defined Literals](./Ders7) | `Ders7/` | Perfect returning (`decltype(auto)`), lambda'da perfect returning, `decltype(auto)` return type quiz (parantez tuzağı, ternary, xvalue member access), universal reference overload önceliği, `std::ref` / `std::reference_wrapper`, tag dispatch (empty class, `constexpr` tag nesneleri), user-defined literals (literal operator function, cooked/raw, suffix kuralları, strong typing, standart kütüphane UDL'leri) |
| 8 | [UDL Template, Strong Types, `std::string_view` ve `std::optional` Giriş](./Ders8) | `Ders8/` | Literal operator template (variadic `char` pack), strong typing (zero-cost abstraction), `std::string_view` (non-owning observer, constructorlar, dangling pointer, null-termination tuzağı, `remove_prefix`/`remove_suffix`, `substr` O(1), `constexpr`, fonksiyon parametresi), `std::optional` giriş (nullable value, `nullopt`, `value_or`, monadic interface C++23) |
