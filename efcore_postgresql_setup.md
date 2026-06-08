# Konfiguracja Entity Framework Core z PostgreSQL (.NET 10)

Ten przewodnik krok po kroku opisuje, jak zainstalować i skonfigurować Entity Framework Core do obsługi bazy danych PostgreSQL w Twoim projekcie Blazor/ASP.NET Core, odwzorowując strukturę tabel z Twojego diagramu ERD.

---

## Krok 1: Instalacja Pakietów NuGet

W katalogu projektu (tam, gdzie znajduje się plik `.csproj`, np. w folderze `jira`) należy uruchomić poniższe komendy w terminalu:

```bash
# Dodanie dostawcy PostgreSQL dla EF Core
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL

# Dodanie narzędzi do obsługi migracji (narzędzia czasu projektowania)
dotnet add package Microsoft.EntityFrameworkCore.Design
```

Jeśli nie masz jeszcze zainstalowanego globalnego narzędzia interfejsu wiersza poleceń EF Core (`dotnet-ef`), zainstaluj je globalnie na swoim systemie:
```bash
dotnet tool install --global dotnet-ef
```
*(Jeśli już je masz, możesz je zaktualizować komendą `dotnet tool update --global dotnet-ef`)*

---

## Krok 2: Definicja Klas Modelu (Entities)

Poniżej znajdują się definicje klas C# odwzorowujące diagram ERD. Używamy atrybutów lub konfiguracji Fluent API w celu zapewnienia, że nazwy tabel i kolumn będą zgodne z diagramem (wielkie litery) oraz będą posiadały odpowiednie typy danych.

Tworzymy nowy folder w projekcie, np. `Models` (lub `Entities`), i umieszczamy w nim poniższe klasy:

### 1. Uzytkownik.cs
```csharp
using System;
using System.Collections.Generic;

namespace jira.Models;

public class Uzytkownik
{
    public int IdUzytkownika { get; set; }
    public string Email { get; set; } = null!;
    public string PasswordHash { get; set; } = null!;
    public string NazwaUzytkownika { get; set; } = null!;
    public string? AvatarUrl { get; set; }
    public string? GithubId { get; set; }
    public string? GoogleId { get; set; }
    public string? GithubRefreshTokenEncrypted { get; set; }
    public string? GoogleRefreshTokenEncrypted { get; set; }
    public DateTime DataRejestracji { get; set; } = DateTime.UtcNow;

    // Relacje (Navigation properties)
    public ICollection<Tablica> TabliceOwner { get; set; } = new List<Tablica>();
    public ICollection<TablicaUzytkownik> TabliceUzyt { get; set; } = new List<TablicaUzytkownik>();
    public ICollection<Zadanie> ZadaniaStworzone { get; set; } = new List<Zadanie>();
    public ICollection<Zadanie> ZadaniaPrzypisane { get; set; } = new List<Zadanie>();
    public ICollection<Komentarz> Komentarze { get; set; } = new List<Komentarz>();
}
```

### 2. Tablica.cs
```csharp
using System;
using System.Collections.Generic;

namespace jira.Models;

public class Tablica
{
    public int IdTablicy { get; set; }
    public string NazwaTablicy { get; set; } = null!;
    public string? OpisTablicy { get; set; }
    public int IdUzytkownikaOwner { get; set; }
    public DateTime DataStworzenia { get; set; } = DateTime.UtcNow;
    public string? KolorTablicy { get; set; }

    // Relacje
    public Uzytkownik Owner { get; set; } = null!;
    public ICollection<TablicaUzytkownik> TabliceUzyt { get; set; } = new List<TablicaUzytkownik>();
    public ICollection<Zadanie> Zadania { get; set; } = new List<Zadanie>();
}
```

### 3. TablicaUzytkownik.cs (Tabela łącząca UZYTKOWNICY <-> TABLICE)
```csharp
using System;

namespace jira.Models;

public class TablicaUzytkownik
{
    public int IdUzytkownika { get; set; }
    public int IdTablicy { get; set; }
    public string Rola { get; set; } = "member"; // np. "admin", "member", "viewer"
    public DateTime DataDolaczenia { get; set; } = DateTime.UtcNow;

    // Relacje
    public Uzytkownik Uzytkownik { get; set; } = null!;
    public Tablica Tablica { get; set; } = null!;
}
```

### 4. Zadanie.cs
```csharp
using System;
using System.Collections.Generic;

namespace jira.Models;

public class Zadanie
{
    public int IdZadania { get; set; }
    public int IdTablicy { get; set; }
    public string TytulZadania { get; set; } = null!;
    public string? OpisZadania { get; set; }
    public DateTime DataStworzenia { get; set; } = DateTime.UtcNow;
    public int? IdUzytkownikaPrzypisanego { get; set; }
    public int IdUzytkownikaTworcyZadania { get; set; }
    public string Priorytet { get; set; } = "sredni"; // "wysoki", "sredni", "niski"
    public string Status { get; set; } = "Todo"; // "Todo", "In Progress", "In Review", "Done"
    public DateTime? DataZakonczenia { get; set; }
    public string KolumnaTablicy { get; set; } = "Todo";

    // Relacje
    public Tablica Tablica { get; set; } = null!;
    public Uzytkownik? UzytkownikPrzypisany { get; set; }
    public Uzytkownik TworcaZadania { get; set; } = null!;
    public ICollection<Komentarz> Komentarze { get; set; } = new List<Komentarz>();
}
```

### 5. Komentarz.cs
```csharp
using System;

namespace jira.Models;

public class Komentarz
{
    public int IdKomentarza { get; set; }
    public int IdZadania { get; set; }
    public string TrescKomentarza { get; set; } = null!;
    public int IdUzytkownika { get; set; }
    public DateTime DataUtworzenia { get; set; } = DateTime.UtcNow;
    public DateTime? DataEdycji { get; set; }

    // Relacje
    public Zadanie Zadanie { get; set; } = null!;
    public Uzytkownik Uzytkownik { get; set; } = null!;
}
```

---

## Krok 3: Konfiguracja Contextu Bazy Danych (`AppDbContext.cs`)

Context bazy danych definiuje mapowanie Fluent API. Konfigurujemy klucze główne, obce, unikalne (np. unikalny email użytkownika) oraz mapujemy nazwy klas i właściwości na nazwy tabel i kolumn z diagramu ERD (np. `UZYTKOWNICY`, `id_uzytkownika` itd.).

Stwórz klasę `AppDbContext.cs` w folderze `Data` lub głównym katalogu projektu:

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

        // --- UZYTKOWNICY ---
        modelBuilder.Entity<Uzytkownik>(entity =>
        {
            entity.ToTable("UZYTKOWNICY");
            entity.HasKey(e => e.IdUzytkownika);
            entity.Property(e => e.IdUzytkownika).HasColumnName("id_uzytkownika");
            
            entity.HasIndex(e => e.Email).IsUnique();
            entity.Property(e => e.Email).HasColumnName("email").IsRequired();
            
            entity.Property(e => e.PasswordHash).HasColumnName("password_hash").IsRequired();
            entity.Property(e => e.NazwaUzytkownika).HasColumnName("nazwa_uzytkownika").IsRequired();
            entity.Property(e => e.AvatarUrl).HasColumnName("avatar_url");
            entity.Property(e => e.GithubId).HasColumnName("github_id");
            entity.Property(e => e.GoogleId).HasColumnName("google_id");
            entity.Property(e => e.GithubRefreshTokenEncrypted).HasColumnName("github_refresh_token_encrypted");
            entity.Property(e => e.GoogleRefreshTokenEncrypted).HasColumnName("google_refresh_token_encrypted");
            entity.Property(e => e.DataRejestracji).HasColumnName("data_rejestracji").HasColumnType("timestamp without time zone");
        });

        // --- TABLICE ---
        modelBuilder.Entity<Tablica>(entity =>
        {
            entity.ToTable("TABLICE");
            entity.HasKey(e => e.IdTablicy);
            entity.Property(e => e.IdTablicy).HasColumnName("id_tablicy");
            
            entity.Property(e => e.NazwaTablicy).HasColumnName("nazwa_tablicy").IsRequired();
            entity.Property(e => e.OpisTablicy).HasColumnName("opis_tablicy");
            entity.Property(e => e.IdUzytkownikaOwner).HasColumnName("id_uzytkownika_owner");
            entity.Property(e => e.DataStworzenia).HasColumnName("data_stworzenia").HasColumnType("timestamp without time zone");
            entity.Property(e => e.KolorTablicy).HasColumnName("kolor_tablicy");

            // Relacja ownera
            entity.HasOne(d => d.Owner)
                .WithMany(p => p.TabliceOwner)
                .HasForeignKey(d => d.IdUzytkownikaOwner)
                .OnDelete(DeleteBehavior.Restrict);
        });

        // --- TABLICE_UZYTKOWNICY ---
        modelBuilder.Entity<TablicaUzytkownik>(entity =>
        {
            entity.ToTable("TABLICE_UZYTKOWNICY");
            entity.HasKey(e => new { e.IdUzytkownika, e.IdTablicy });
            
            entity.Property(e => e.IdUzytkownika).HasColumnName("id_uzytkownika");
            entity.Property(e => e.IdTablicy).HasColumnName("id_tablicy");
            entity.Property(e => e.Rola).HasColumnName("rola").IsRequired();
            entity.Property(e => e.DataDolaczenia).HasColumnName("data_dolaczenia").HasColumnType("timestamp without time zone");

            // Relacje klucza złożonego
            entity.HasOne(d => d.Uzytkownik)
                .WithMany(p => p.TabliceUzyt)
                .HasForeignKey(d => d.IdUzytkownika)
                .OnDelete(DeleteBehavior.Cascade);

            entity.HasOne(d => d.Tablica)
                .WithMany(p => p.TabliceUzyt)
                .HasForeignKey(d => d.IdTablicy)
                .OnDelete(DeleteBehavior.Cascade);
        });

        // --- ZADANIA ---
        modelBuilder.Entity<Zadanie>(entity =>
        {
            entity.ToTable("ZADANIA");
            entity.HasKey(e => e.IdZadania);
            entity.Property(e => e.IdZadania).HasColumnName("id_zadania");
            
            entity.Property(e => e.IdTablicy).HasColumnName("id_tablicy");
            entity.Property(e => e.TytulZadania).HasColumnName("tytul_zadania").IsRequired();
            entity.Property(e => e.OpisZadania).HasColumnName("opis_zadania");
            entity.Property(e => e.DataStworzenia).HasColumnName("data_stworzenia").HasColumnType("timestamp without time zone");
            entity.Property(e => e.IdUzytkownikaPrzypisanego).HasColumnName("id_uzytkownika_przypisanego");
            entity.Property(e => e.IdUzytkownikaTworcyZadania).HasColumnName("id_uzytkownika_tworcy_zadania");
            entity.Property(e => e.Priorytet).HasColumnName("priorytet").IsRequired();
            entity.Property(e => e.Status).HasColumnName("status").IsRequired();
            entity.Property(e => e.DataZakonczenia).HasColumnName("data_zakonczenia").HasColumnType("timestamp without time zone");
            entity.Property(e => e.KolumnaTablicy).HasColumnName("kolumna_tablicy").IsRequired();

            // Relacje
            entity.HasOne(d => d.Tablica)
                .WithMany(p => p.Zadania)
                .HasForeignKey(d => d.IdTablicy)
                .OnDelete(DeleteBehavior.Cascade);

            entity.HasOne(d => d.UzytkownikPrzypisany)
                .WithMany(p => p.ZadaniaPrzypisane)
                .HasForeignKey(d => d.IdUzytkownikaPrzypisanego)
                .OnDelete(DeleteBehavior.SetNull);

            entity.HasOne(d => d.TworcaZadania)
                .WithMany(p => p.ZadaniaStworzone)
                .HasForeignKey(d => d.IdUzytkownikaTworcyZadania)
                .OnDelete(DeleteBehavior.Restrict);
        });

        // --- KOMENTARZE ---
        modelBuilder.Entity<Komentarz>(entity =>
        {
            entity.ToTable("KOMENTARZE");
            entity.HasKey(e => e.IdKomentarza);
            entity.Property(e => e.IdKomentarza).HasColumnName("id_komentarza");
            
            entity.Property(e => e.IdZadania).HasColumnName("id_zadania");
            entity.Property(e => e.TrescKomentarza).HasColumnName("tresc_komentarza").IsRequired();
            entity.Property(e => e.IdUzytkownika).HasColumnName("id_uzytkownika");
            entity.Property(e => e.DataUtworzenia).HasColumnName("data_utworzenia").HasColumnType("timestamp without time zone");
            entity.Property(e => e.DataEdycji).HasColumnName("data_edycji").HasColumnType("timestamp without time zone");

            // Relacje
            entity.HasOne(d => d.Zadanie)
                .WithMany(p => p.Komentarze)
                .HasForeignKey(d => d.IdZadania)
                .OnDelete(DeleteBehavior.Cascade);

            entity.HasOne(d => d.Uzytkownik)
                .WithMany(p => p.Komentarze)
                .HasForeignKey(d => d.IdUzytkownika)
                .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

---

## Krok 4: Rejestracja DbContext w `Program.cs`

Musimy zarejestrować `AppDbContext` w kontenerze Dependency Injection aplikacji. W pliku `Program.cs` (w okolicy linii 5-7, przed wywołaniem `builder.Build()`) należy dodać:

```csharp
using Microsoft.EntityFrameworkCore;
using jira.Data;

// ...

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
```

---

## Krok 5: Konfiguracja Connection Stringa

### Ustawienia Lokalne (`appsettings.Development.json` lub `appsettings.json`)
Dodaj sekcję `ConnectionStrings` ze szczegółami połączenia do PostgreSQL. Ponieważ z `.env` wynika, że Twoje dane logowania to użytkownik: `root`, hasło: `maslo`, baza: `jira_db`, a serwer jest dostępny na porcie `5432`, konfiguracja wygląda następująco:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=jira_db;Username=root;Password=maslo"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

### Konfiguracja Docker
Gdy aplikacja i baza danych działają w Dockerze (`docker-compose.yml`), connection string jest automatycznie nadpisywany zmienną środowiskową:
`ConnectionStrings__DefaultConnection: "Host=db;Database=${POSTGRES_DB};Username=${POSTGRES_USER};Password=${POSTGRES_PASSWORD}"`
EF Core automatycznie dopasuje ten format w kontenerze.

---

## Krok 6: Generowanie i Nakładanie Migracji

Po poprawnym skompilowaniu projektu (`dotnet build`), możesz utworzyć pierwszą migrację i zaaplikować ją do bazy danych:

1. **Utworzenie migracji:**
   ```bash
   dotnet ef migrations add InitialCreate
   ```
   *Spowoduje to utworzenie folderu `Migrations` w projekcie z kodem C# opisującym strukturę tabel.*

2. **Zastosowanie migracji do bazy danych (uruchomienie lokalnie):**
   ```bash
   dotnet ef database update
   ```
   *Ta komenda połączy się z lokalną bazą danych PostgreSQL na podstawie connection stringa i utworzy wszystkie tabele.*
