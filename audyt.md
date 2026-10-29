# Audyt bezpieczeństwa plików

System: WSL, Ubuntu 22.04.
`2>/dev/null` ukrywa błędy "Permission denied", `-xdev` nie wchodzi na inne dyski.

## 1. Pliki z SUID należące do root

```bash
sudo find / -xdev -type f -perm -4000 -user root 2>/dev/null
sudo find / -xdev -type f -perm -4000 -user root 2>/dev/null | wc -l
```

Liczba: **21**

```
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/su
/usr/bin/mount
/usr/lib/virtualbox/VirtualBoxVM
/usr/lib/virtualbox/VBoxHeadless
...
```

Ocena: normalne. `passwd`, `sudo`, `su`, `mount` muszą mieć SUID, bo zwykły
użytkownik robi nimi rzeczy, do których potrzebny jest roo. Do sprawdzenia: 6 plików z `/usr/lib/virtualbox/` - zostały
po VirtualBoxie zainstalowanym w WSL, który nie działa.

## 2. Katalogi zapisywalne dla wszystkich (bez /tmp i /var/tmp)

```bash
sudo find / -xdev -type d -perm -o+w ! -path '/tmp*' ! -path '/var/tmp*' 2>/dev/null
ls -ld /var/crash /mnt/c
```

Liczba: **5**

```
/var/crash
/mnt/wsl
/mnt/d
/mnt/wslg
/mnt/c
```

```
drwxrwxrwx 1 krywapo krywapo 4096 Oct 29 11:15 /mnt/c
drwxrwxrwt 2 root    root    4096 Oct 15 18:51 /var/crash
```

Ocena: normalne.
- `/var/crash` ma bit sticky (`t` na końcu) - każdy może tu zapisać, ale usunąć
  może tylko swoje pliki, tak jak w `/tmp`.
- `/mnt/c` i `/mnt/d` to dyski Windowsa. WSL pokazuje je jako 777, a o dostępie
  decyduje Windows.
- `/mnt/wsl` i `/mnt/wslg` to katalogi samego WSL.

## 3. Pliki większe niż 50 MB w /var

```bash
sudo find /var -xdev -type f -size +50M -exec ls -lh {} +
sudo find /var -xdev -type f -size +50M 2>/dev/null | wc -l
```

Liczba: **12**

```
68M  /var/cache/apt/pkgcache.bin
68M  /var/cache/apt/srcpkgcache.bin
62M  /var/lib/apt/lists/archive.ubuntu.com_ubuntu_dists_jammy_universe_binary-amd64_Packages
132M /var/lib/snapd/snaps/ubuntu-desktop-installer_1286.snap
74M  /var/lib/snapd/snaps/core22_2955.snap
...
```

Ocena: normalne. 3 pliki to cache i lista pakietów `apt`, 9 plików to pakiety
`snap`. Nie ma dużych logów ani obcych plików.

## 4. Pliki niezmieniane od ponad 365 dni w katalogu domowym

```bash
find ~ -type f -mtime +365 2>/dev/null | wc -l
find ~ -type f -mtime +365 2>/dev/null | cut -d/ -f4 | sort | uniq -c | sort -rn | head
```

Liczba: **25721**

```
  24169 .cache
   1548 .npm
      1 .yarnrc
      1 .sudo_as_admin_successful
      1 .profile
      1 .bash_logout
```

Prawie wszystko to `.cache` i `.npm` - pamięć podręczna programów
(`.npm` od Node.js). Można to usunąć. Pozostałe to pliki startowe powłoki, które się nie zmieniają.

## Podsumowanie

| Sekcja | Liczba | Ocena |
|---|---|---|
| SUID root | 21 | normalne, do usunięcia VirtualBox w WSL |
| Katalogi zapisywalne dla wszystkich | 5 | normalne (sticky lub dyski Windowsa) |
| Pliki > 50 MB w /var | 12 | normalne (apt i snap) |
| Pliki > 365 dni w ~ | 25721 | normalne (cache) |
