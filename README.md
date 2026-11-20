# Zadanie Ansible · wariant A

Rola `webserver` konfiguruje nginx i konto `deploy` na dedykowanym hoście Rocky Linux 9 lub AlmaLinux 9. Inventory rozdziela `dev` (TCP 8008) i `prod` (TCP 80). Pliki w `inventory/group_vars/` definiują także nazwę środowiska i `welcome_message`.

## Założenia

- Kontroler z `ansible-core` 2.19 (obsługuje systemowy Python 3.9 na EL9), dostępem SSH do hostów i kluczem konta administracyjnego. Konto administracyjne może podnieść uprawnienia przez sudo.
- Hosty mają systemd, działający sshd z obsługą kluczy publicznych, Python 3 z bibliotekami systemowymi DNF i dostęp do repozytorium z pakietem nginx.
- Rola zarządza całym `/etc/nginx/nginx.conf` i całym plikiem `/home/deploy/.ssh/authorized_keys`. Konto `deploy` ma zablokowane hasło i nie otrzymuje uprawnień sudo.
- Zapora i routing są konfigurowane poza rolą. Dla dostępu z innej maszyny wybrany port musi być dozwolony na zaporze. Domyślne porty 80 i 8008 są objęte typową polityką SELinux dla HTTP na EL9. Przy zmianie portu sprawdź również jego typ SELinux.
- Serwis udostępnia stronę przez HTTP. W tym zadaniu grupa `prod` identyfikuje środowisko w inventory.

## Uruchomienie

1. Rozpakuj archiwum i przejdź do katalogu z `playbook.yml`.
2. W `inventory/hosts.yml` zastąp przykładowe adresy adresami własnych hostów. Opcjonalnie zmień porty i powitania w plikach `group_vars`.
3. Wskaż plik **publicznego** klucza użytkownika `deploy` i uruchom playbook. W przykładzie `admin` to konto administracyjne na hoście.

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml --syntax-check
DEPLOY_PUBKEY="$HOME/.ssh/deploy_ed25519.pub"
DEPLOY_SSH_PUBLIC_KEY="$(cat "$DEPLOY_PUBKEY")" \
  ansible-playbook -i inventory/hosts.yml playbook.yml --limit dev -u admin
```

Dla środowiska produkcyjnego użyj `--limit prod`. Jeśli sudo wymaga hasła, dodaj `--ask-become-pass`.

## Weryfikacja

- Uruchom ponownie tę samą komendę bez zmian na hoście ani w zmiennych. Oczekiwany wynik w `PLAY RECAP` to `changed=0`, `failed=0`, `unreachable=0`.
- Na hoście sprawdź `systemctl is-active nginx` oraz `systemctl is-enabled nginx`. Oczekiwane odpowiedzi to `active` i `enabled`.
- Na hoście dev wykonaj `curl -fsS http://127.0.0.1:8008/`. Na hoście prod wykonaj `curl -fsS http://127.0.0.1/`. Strona powinna pokazać hostname, środowisko i powitanie.
- Z kontrolera sprawdź logowanie poleceniem `ssh -o IdentitiesOnly=yes -i ~/.ssh/deploy_ed25519 deploy@ADRES_HOSTA id -un`. Oczekiwany wynik to `deploy`.
- Zmień wyłącznie `welcome_message` i uruchom playbook. Powinna zmienić się strona, a handler nginx nie powinien się uruchomić. Zmiana portu na wolny i dozwolony przez SELinux powinna uruchomić handler.

Szablon konfiguracji zawiera osobny szablon vhosta. Ansible sprawdza całą konfigurację poleceniem `nginx -t` przed zastąpieniem pliku. Tylko zadanie wdrażające konfigurację może powiadomić handler, więc sama aktualizacja HTML lub klucza SSH nie przeładowuje usługi.
