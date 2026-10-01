# Part 17: Optional & Date/Time API
## ขั้นตอนที่ 1081-1140: จัดการ Null Safety และวันเวลา

---

## 17.1 Optional<T>

```java
import java.util.*;
import java.util.stream.*;

public class OptionalDemo {
    
    record User(String name, String email, Address address) {}
    record Address(String street, String city, String zip) {}
    
    // Without Optional (null checks everywhere)
    static String getCityOld(User user) {
        if (user != null && user.address() != null) {
            return user.address().city();
        }
        return "Unknown";
    }
    
    // With Optional (chained)
    static String getCityNew(User user) {
        return Optional.ofNullable(user)
                       .map(User::address)
                       .map(Address::city)
                       .orElse("Unknown");
    }
    
    // Repository simulation
    static final Map<Integer, User> userDB = Map.of(
        1, new User("Alice", "alice@example.com",
                    new Address("123 Main St", "Bangkok", "10100")),
        2, new User("Bob", "bob@example.com", null),
        3, new User("Charlie", null, new Address("456 Oak Ave", "Chiang Mai", "50000"))
    );
    
    static Optional<User> findById(int id) {
        return Optional.ofNullable(userDB.get(id));
    }
    
    static Optional<String> findEmail(int userId) {
        return findById(userId)
               .flatMap(u -> Optional.ofNullable(u.email()));
    }
    
    public static void main(String[] args) {
        
        // Creating Optional
        Optional<String> present = Optional.of("Hello");
        Optional<String> empty = Optional.empty();
        Optional<String> nullable = Optional.ofNullable(null);
        
        System.out.println("present: " + present);
        System.out.println("empty: " + empty);
        System.out.println("nullable: " + nullable);
        
        // Checking
        System.out.println("\nisPresent: " + present.isPresent());
        System.out.println("isEmpty: " + empty.isEmpty());
        
        // Getting value
        System.out.println("\nget: " + present.get());
        System.out.println("orElse: " + empty.orElse("default"));
        System.out.println("orElseGet: " + empty.orElseGet(() -> "computed default"));
        System.out.println("orElseThrow: " + present.orElseThrow());
        
        // Transforming
        System.out.println("\nmap: " + present.map(String::toUpperCase));
        System.out.println("map empty: " + empty.map(String::toUpperCase));
        System.out.println("filter: " + present.filter(s -> s.length() > 3));
        
        // Executing
        present.ifPresent(v -> System.out.println("\nifPresent: " + v));
        empty.ifPresentOrElse(
            v -> System.out.println("present: " + v),
            () -> System.out.println("empty! using default")
        );
        
        // Optional chaining
        System.out.println("\n=== User lookups ===");
        for (int id : new int[]{1, 2, 3, 4}) {
            String city = findById(id)
                .map(User::address)
                .map(Address::city)
                .orElse("No address");
            System.out.printf("User %d city: %s%n", id, city);
        }
        
        // Email lookups
        System.out.println("\n=== Email lookups ===");
        for (int id : new int[]{1, 2, 3, 4}) {
            String email = findEmail(id).orElse("No email");
            System.out.printf("User %d email: %s%n", id, email);
        }
        
        // Optional with Stream
        System.out.println("\n=== Optional with Stream ===");
        List<Optional<String>> opts = List.of(
            Optional.of("Java"), Optional.empty(), Optional.of("Kotlin"),
            Optional.empty(), Optional.of("Python")
        );
        
        // Collect only present values (Java 9+)
        List<String> presents = opts.stream()
            .flatMap(Optional::stream)
            .collect(Collectors.toList());
        System.out.println("Present: " + presents);
        
        // or() - provide alternative Optional (Java 9+)
        Optional<String> result = empty.or(() -> Optional.of("fallback value"));
        System.out.println("or(): " + result);
    }
}
```

---

## 17.2 Date/Time API (java.time)

```java
import java.time.*;
import java.time.format.*;
import java.time.temporal.*;

public class DateTimeDemo {
    
    public static void main(String[] args) {
        
        // ====== LocalDate ======
        System.out.println("=== LocalDate ===");
        LocalDate today = LocalDate.now();
        LocalDate newYear = LocalDate.of(2024, 1, 1);
        LocalDate birthday = LocalDate.of(1990, Month.MARCH, 15);
        
        System.out.println("Today: " + today);
        System.out.println("New Year: " + newYear);
        System.out.println("Birthday: " + birthday);
        
        // Manipulation
        LocalDate tomorrow = today.plusDays(1);
        LocalDate nextMonth = today.plusMonths(1);
        LocalDate lastYear = today.minusYears(1);
        System.out.println("Tomorrow: " + tomorrow);
        System.out.println("Next month: " + nextMonth);
        System.out.println("Last year: " + lastYear);
        
        // Extracting
        System.out.println("Year: " + today.getYear());
        System.out.println("Month: " + today.getMonth());
        System.out.println("Day of week: " + today.getDayOfWeek());
        System.out.println("Day of year: " + today.getDayOfYear());
        
        // Comparing
        System.out.println("today after newYear: " + today.isAfter(newYear));
        System.out.println("birthday before today: " + birthday.isBefore(today));
        
        // ====== LocalTime ======
        System.out.println("\n=== LocalTime ===");
        LocalTime now = LocalTime.now();
        LocalTime lunch = LocalTime.of(12, 30, 0);
        LocalTime midnight = LocalTime.MIDNIGHT;
        
        System.out.println("Now: " + now);
        System.out.println("Lunch: " + lunch);
        System.out.println("Midnight: " + midnight);
        
        LocalTime later = now.plusHours(2).plusMinutes(30);
        System.out.println("2h30m later: " + later);
        System.out.println("Hour: " + now.getHour());
        System.out.println("is before lunch: " + now.isBefore(lunch));
        
        // ====== LocalDateTime ======
        System.out.println("\n=== LocalDateTime ===");
        LocalDateTime datetime = LocalDateTime.now();
        LocalDateTime specific = LocalDateTime.of(2024, 6, 15, 10, 30, 0);
        
        System.out.println("Now: " + datetime);
        System.out.println("Specific: " + specific);
        
        // Combine
        LocalDateTime combined = LocalDateTime.of(today, lunch);
        System.out.println("Today at lunch: " + combined);
        
        // Extract
        LocalDate datePart = datetime.toLocalDate();
        LocalTime timePart = datetime.toLocalTime();
        System.out.println("Date part: " + datePart);
        System.out.println("Time part: " + timePart);
        
        // ====== Period & Duration ======
        System.out.println("\n=== Period & Duration ===");
        Period period = Period.between(birthday, today);
        System.out.printf("Age: %d years, %d months, %d days%n",
            period.getYears(), period.getMonths(), period.getDays());
        
        Period twoWeeks = Period.ofWeeks(2);
        System.out.println("2 weeks from now: " + today.plus(twoWeeks));
        
        Duration duration = Duration.between(LocalTime.of(9, 0), LocalTime.of(17, 30));
        System.out.println("Work hours: " + duration.toHours() + "h " + 
            duration.toMinutesPart() + "m");
        
        Duration elapsed = Duration.ofHours(2).plusMinutes(45).plusSeconds(30);
        System.out.printf("Elapsed: %dh %dm %ds%n",
            elapsed.toHours(), elapsed.toMinutesPart(), elapsed.toSecondsPart());
        
        // ====== ZonedDateTime ======
        System.out.println("\n=== ZonedDateTime ===");
        ZonedDateTime bangkokTime = ZonedDateTime.now(ZoneId.of("Asia/Bangkok"));
        ZonedDateTime tokyoTime = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
        ZonedDateTime newYorkTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
        ZonedDateTime londonTime = ZonedDateTime.now(ZoneId.of("Europe/London"));
        
        DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
        System.out.println("Bangkok:  " + bangkokTime.format(fmt));
        System.out.println("Tokyo:    " + tokyoTime.format(fmt));
        System.out.println("New York: " + newYorkTime.format(fmt));
        System.out.println("London:   " + londonTime.format(fmt));
        
        // ====== Formatting & Parsing ======
        System.out.println("\n=== Formatting & Parsing ===");
        DateTimeFormatter thaiFormatter = DateTimeFormatter.ofPattern("d MMMM yyyy", 
            java.util.Locale.ENGLISH);
        DateTimeFormatter isoFormatter = DateTimeFormatter.ISO_LOCAL_DATE;
        DateTimeFormatter customFmt = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");
        
        System.out.println("Thai format: " + today.format(thaiFormatter));
        System.out.println("ISO format: " + today.format(isoFormatter));
        System.out.println("Custom: " + datetime.format(customFmt));
        
        // Parsing
        LocalDate parsed = LocalDate.parse("15/06/2024",
            DateTimeFormatter.ofPattern("dd/MM/yyyy"));
        System.out.println("Parsed: " + parsed);
        
        LocalDateTime parsedDT = LocalDateTime.parse("2024-06-15T10:30:00");
        System.out.println("Parsed DT: " + parsedDT);
    }
}
```

---

## 17.3 Date/Time Calculations

```java
import java.time.*;
import java.time.temporal.*;
import java.util.*;
import java.util.stream.*;

public class DateTimeCalculations {
    
    // Business days calculator
    static long businessDaysBetween(LocalDate start, LocalDate end) {
        return start.datesUntil(end)
                    .filter(d -> d.getDayOfWeek() != DayOfWeek.SATURDAY &&
                                 d.getDayOfWeek() != DayOfWeek.SUNDAY)
                    .count();
    }
    
    // Next working day
    static LocalDate nextWorkingDay(LocalDate date) {
        LocalDate next = date.plusDays(1);
        while (next.getDayOfWeek() == DayOfWeek.SATURDAY ||
               next.getDayOfWeek() == DayOfWeek.SUNDAY) {
            next = next.plusDays(1);
        }
        return next;
    }
    
    // Working days in a month
    static List<LocalDate> workingDaysInMonth(int year, int month) {
        return LocalDate.of(year, month, 1)
                        .datesUntil(LocalDate.of(year, month, 1).plusMonths(1))
                        .filter(d -> d.getDayOfWeek() != DayOfWeek.SATURDAY &&
                                     d.getDayOfWeek() != DayOfWeek.SUNDAY)
                        .collect(Collectors.toList());
    }
    
    // Age calculator
    static String calculateAge(LocalDate birthDate) {
        Period period = Period.between(birthDate, LocalDate.now());
        return period.getYears() + " years, " + period.getMonths() + " months, " + 
               period.getDays() + " days";
    }
    
    // Countdown timer
    static String countdown(LocalDate targetDate) {
        LocalDate today = LocalDate.now();
        if (targetDate.isBefore(today)) return "Already passed!";
        if (targetDate.equals(today)) return "TODAY!";
        
        Period p = Period.between(today, targetDate);
        long totalDays = ChronoUnit.DAYS.between(today, targetDate);
        
        if (p.getYears() > 0) {
            return String.format("%d years, %d months, %d days (%d total days)",
                p.getYears(), p.getMonths(), p.getDays(), totalDays);
        } else if (p.getMonths() > 0) {
            return String.format("%d months, %d days (%d total days)",
                p.getMonths(), p.getDays(), totalDays);
        } else {
            return totalDays + " days";
        }
    }
    
    public static void main(String[] args) {
        
        LocalDate start = LocalDate.of(2024, 1, 1);
        LocalDate end = LocalDate.of(2024, 3, 31);
        
        System.out.println("Business days (Q1 2024): " + businessDaysBetween(start, end));
        
        LocalDate friday = LocalDate.of(2024, 5, 31);  // Friday
        System.out.println("Next working day after " + friday + ": " + nextWorkingDay(friday));
        
        List<LocalDate> workDays = workingDaysInMonth(2024, 1);
        System.out.println("Working days in Jan 2024: " + workDays.size());
        
        LocalDate birthDate = LocalDate.of(1995, 8, 20);
        System.out.println("Age: " + calculateAge(birthDate));
        
        System.out.println("Countdown to 2025-01-01: " +
            countdown(LocalDate.of(2025, 1, 1)));
        System.out.println("Countdown to 2024-12-25: " +
            countdown(LocalDate.of(2024, 12, 25)));
        
        // Calendar view (current month)
        System.out.println("\n=== Calendar View ===");
        LocalDate now = LocalDate.now();
        LocalDate firstOfMonth = now.withDayOfMonth(1);
        LocalDate lastOfMonth = now.withDayOfMonth(now.lengthOfMonth());
        
        System.out.println("     " + now.getMonth() + " " + now.getYear());
        System.out.println("Mo Tu We Th Fr Sa Su");
        
        // Padding
        int firstWeekDay = firstOfMonth.getDayOfWeek().getValue(); // 1=Mon, 7=Sun
        System.out.print("   ".repeat(firstWeekDay - 1));
        
        for (LocalDate d = firstOfMonth; !d.isAfter(lastOfMonth); d = d.plusDays(1)) {
            boolean isToday = d.equals(now);
            System.out.printf("%s%2d%s", isToday ? "[" : " ", d.getDayOfMonth(), isToday ? "]" : " ");
            if (d.getDayOfWeek() == DayOfWeek.SUNDAY) System.out.println();
        }
        System.out.println();
        
        // Instant (unix timestamp)
        System.out.println("\n=== Instant ===");
        Instant instant = Instant.now();
        System.out.println("Epoch seconds: " + instant.getEpochSecond());
        System.out.println("Millis: " + instant.toEpochMilli());
        
        Instant twoHoursAgo = instant.minus(2, ChronoUnit.HOURS);
        Duration diff = Duration.between(twoHoursAgo, instant);
        System.out.println("Diff: " + diff.toHours() + "h " + diff.toMinutesPart() + "m");
    }
}
```

---

## สรุป Part 17

| API | หน้าที่ |
|-----|---------|
| `Optional<T>` | ป้องกัน NullPointerException แบบ functional |
| `LocalDate` | วันที่ (ไม่มี time zone) |
| `LocalTime` | เวลา (ไม่มี time zone) |
| `LocalDateTime` | วันที่ + เวลา |
| `ZonedDateTime` | วันที่ + เวลา + timezone |
| `Period` | ช่วงเวลาในหน่วย years/months/days |
| `Duration` | ช่วงเวลาในหน่วย hours/minutes/seconds |
| `Instant` | Unix timestamp |
| `DateTimeFormatter` | Format/Parse string ↔ datetime |

➡️ [Part 18: Design Patterns](./Part-18-Design-Patterns.md)
