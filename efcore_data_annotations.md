# Podejście Data Annotations w EF Core (Atrybuty `[]`)

W tym podejściu mapowanie tabel, kolumn, kluczy i relacji przenosimy bezpośrednio do klas modeli za pomocą atrybutów w nawiasach kwadratowych (np. `[Table]`, `[Column]`, `[Key]`). Dzięki temu klasa `AppDbContext` staje się niezwykle mała i czysta.

---

## 1. Uproszczony `AppDbContext`

Gdy całe mapowanie kolumn jest zdefiniowane w klasach, Twój `DbContext` kurczy się do minimum:

```csharp
using Microsoft.EntityFrameworkCore;
using jira.Models;

namespace jira.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<Uzytkownik> Uzytkownicy => Set<Uzytkownik>();
    public DbSet<Tablica> Tablice => Set<Tablica>();
    public DbSet<TablicaUzytkownik> TabliceUzytkownicy => Set<TablicaUzytkownik>();
    public DbSet<Zadanie> Zadania => Set<Zadanie>();
    public DbSet<Komentarz> Komentarze => Set<Komentarz>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Fluent API jest potrzebne już tylko do konfiguracji zachowania przy usuwaniu (DeleteBehavior)
        // aby uniknąć konfliktów w bazie danych z cyklami kaskadowymi.

        modelBuilder.Entity<Tablica>()
            .HasOne(d => d.Owner)
            .WithMany(p => p.TabliceOwner)
            .HasForeignKey(d => d.IdUzytkownikaOwner)
            .OnDelete(DeleteBehavior.Restrict);

        modelBuilder.Entity<Zadanie>()
            .HasOne(d => d.TworcaZadania)
            .WithMany(p => p.ZadaniaStworzone)
            .HasForeignKey(d => d.IdUzytkownikaTworcyZadania)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

---

## 2. Klasy Modeli z Atrybutami

Wszystkie klasy korzystają teraz z przestrzeni nazw `System.ComponentModel.DataAnnotations` oraz `System.ComponentModel.DataAnnotations.Schema`.

### Uzytkownik.cs
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using Microsoft.EntityFrameworkCore;

namespace jira.Models;

[Table("UZYTKOWNICY")]
[Index(nameof(Email), IsUnique = true)] // Konfiguracja unikalnego klucza
public class Uzytkownik
{
    [Key]
    [Column("id_uzytkownika")]
    public int IdUzytkownika { get; set; }

    [Required]
    [EmailAddress]
    [Column("email")]
    public string Email { get; set; } = null!;

    [Required]
    [Column("password_hash")]
    public string PasswordHash { get; set; } = null!;

    [Required]
    [Column("nazwa_uzytkownika")]
    public string NazwaUzytkownika { get; set; } = null!;

    [Column("avatar_url")]
    public string? AvatarUrl { get; set; }

    [Column("github_id")]
    public string? GithubId { get; set; }

    [Column("google_id")]
    public string? GoogleId { get; set; }

    [Column("github_refresh_token_encrypted")]
    public string? GithubRefreshTokenEncrypted { get; set; }

    [Column("google_refresh_token_encrypted")]
    public string? GoogleRefreshTokenEncrypted { get; set; }

    [Column("data_rejestracji", TypeName = "timestamp without time zone")]
    public DateTime DataRejestracji { get; set; } = DateTime.UtcNow;

    // Relacje
    [InverseProperty(nameof(Tablica.Owner))]
    public ICollection<Tablica> TabliceOwner { get; set; } = new List<Tablica>();

    public ICollection<TablicaUzytkownik> TabliceUzyt { get; set; } = new List<TablicaUzytkownik>();

    [InverseProperty(nameof(Zadanie.TworcaZadania))]
    public ICollection<Zadanie> ZadaniaStworzone { get; set; } = new List<Zadanie>();

    [InverseProperty(nameof(Zadanie.UzytkownikPrzypisany))]
    public ICollection<Zadanie> ZadaniaPrzypisane { get; set; } = new List<Zadanie>();

    public ICollection<Komentarz> Komentarze { get; set; } = new List<Komentarz>();
}
```

### Tablica.cs
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace jira.Models;

[Table("TABLICE")]
public class Tablica
{
    [Key]
    [Column("id_tablicy")]
    public int IdTablicy { get; set; }

    [Required]
    [Column("nazwa_tablicy")]
    public string NazwaTablicy { get; set; } = null!;

    [Column("opis_tablicy")]
    public string? OpisTablicy { get; set; }

    [Column("id_uzytkownika_owner")]
    public int IdUzytkownikaOwner { get; set; }

    [Column("data_stworzenia", TypeName = "timestamp without time zone")]
    public DateTime DataStworzenia { get; set; } = DateTime.UtcNow;

    [Column("kolor_tablicy")]
    public string? KolorTablicy { get; set; }

    // Relacje
    [ForeignKey(nameof(IdUzytkownikaOwner))]
    public Uzytkownik Owner { get; set; } = null!;

    public ICollection<TablicaUzytkownik> TabliceUzyt { get; set; } = new List<TablicaUzytkownik>();
    public ICollection<Zadanie> Zadania { get; set; } = new List<Zadanie>();
}
```

### TablicaUzytkownik.cs (Tabela Łącząca)
Od .NET 7 / EF Core 7 możemy definiować klucz złożony za pomocą atrybutu `[PrimaryKey]` nad klasą:

```csharp
using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using Microsoft.EntityFrameworkCore;

namespace jira.Models;

[Table("TABLICE_UZYTKOWNICY")]
[PrimaryKey(nameof(IdUzytkownika), nameof(IdTablicy))]
public class TablicaUzytkownik
{
    [Column("id_uzytkownika")]
    public int IdUzytkownika { get; set; }

    [Column("id_tablicy")]
    public int IdTablicy { get; set; }

    [Required]
    [Column("rola")]
    public string Rola { get; set; } = "member";

    [Column("data_dolaczenia", TypeName = "timestamp without time zone")]
    public DateTime DataDolaczenia { get; set; } = DateTime.UtcNow;

    // Relacje
    [ForeignKey(nameof(IdUzytkownika))]
    public Uzytkownik Uzytkownik { get; set; } = null!;

    [ForeignKey(nameof(IdTablicy))]
    public Tablica Tablica { get; set; } = null!;
}
```

### Zadanie.cs
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace jira.Models;

[Table("ZADANIA")]
public class Zadanie
{
    [Key]
    [Column("id_zadania")]
    public int IdZadania { get; set; }

    [Column("id_tablicy")]
    public int IdTablicy { get; set; }

    [Required]
    [Column("tytul_zadania")]
    public string TytulZadania { get; set; } = null!;

    [Column("opis_zadania")]
    public string? OpisZadania { get; set; }

    [Column("data_stworzenia", TypeName = "timestamp without time zone")]
    public DateTime DataStworzenia { get; set; } = DateTime.UtcNow;

    [Column("id_uzytkownika_przypisanego")]
    public int? IdUzytkownikaPrzypisanego { get; set; }

    [Column("id_uzytkownika_tworcy_zadania")]
    public int IdUzytkownikaTworcyZadania { get; set; }

    [Required]
    [Column("priorytet")]
    public string Priorytet { get; set; } = "sredni";

    [Required]
    [Column("status")]
    public string Status { get; set; } = "Todo";

    [Column("data_zakonczenia", TypeName = "timestamp without time zone")]
    public DateTime? DataZakonczenia { get; set; }

    [Required]
    [Column("kolumna_tablicy")]
    public string KolumnaTablicy { get; set; } = "Todo";

    // Relacje
    [ForeignKey(nameof(IdTablicy))]
    public Tablica Tablica { get; set; } = null!;

    [ForeignKey(nameof(IdUzytkownikaPrzypisanego))]
    public Uzytkownik? UzytkownikPrzypisany { get; set; }

    [ForeignKey(nameof(IdUzytkownikaTworcyZadania))]
    public Uzytkownik TworcaZadania { get; set; } = null!;

    public ICollection<Komentarz> Komentarze { get; set; } = new List<Komentarz>();
}
```

### Komentarz.cs
```csharp
using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace jira.Models;

[Table("KOMENTARZE")]
public class Komentarz
{
    [Key]
    [Column("id_komentarza")]
    public int IdKomentarza { get; set; }

    [Column("id_zadania")]
    public int IdZadania { get; set; }

    [Required]
    [Column("tresc_komentarza")]
    public string TrescKomentarza { get; set; } = null!;

    [Column("id_uzytkownika")]
    public int IdUzytkownika { get; set; }

    [Column("data_utworzenia", TypeName = "timestamp without time zone")]
    public DateTime DataUtworzenia { get; set; } = DateTime.UtcNow;

    [Column("data_edycji", TypeName = "timestamp without time zone")]
    public DateTime? DataEdycji { get; set; }

    // Relacje
    [ForeignKey(nameof(IdZadania))]
    public Zadanie Zadanie { get; set; } = null!;

    [ForeignKey(nameof(IdUzytkownika))]
    public Uzytkownik Uzytkownik { get; set; } = null!;
}
```
