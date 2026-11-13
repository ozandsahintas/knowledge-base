# Git: "Username for 'https://github.com'" sorusu ve kimlik doğrulama

`git pull` ya da `git push` sırasında şu soru çıkıyor:

```
Username for 'https://github.com':
```

Akla ilk gelen soru: "Git config'imde adım ve e-postam var, neden oradan almıyor?"

## Kısa cevap

`user.name` ve `user.email` **kimlik doğrulama için kullanılmaz.** Yalnızca commit'in
"yazar" bilgisi olarak damgalanır. GitHub'a bağlanırken "bu kişi kim ve yetkili mi?"
sorusunu **credential helper** (kimlik bilgisi yardımcısı) ya da **SSH anahtarı** cevaplar.

| Ayar | Ne işe yarar | Kimlik doğrular mı? |
|---|---|---|
| `user.name`, `user.email` | Commit yazarı | Hayır |
| `credential.helper` | HTTPS'te kullanıcı adı ve token'ı saklar/getirir | Evet (HTTPS) |
| SSH anahtarı (`~/.ssh`) | SSH üzerinden kimlik kanıtı | Evet (SSH) |

## Neden soruyor? Teşhis

Önce remote'un hangi protokolü kullandığına bak:

```bash
git remote -v
```

- `https://github.com/...` → credential helper devreye girer.
- `git@github.com:...` → SSH anahtarı devreye girer, kullanıcı adı sorulmaz.

HTTPS ise hangi helper'ın tanımlı olduğuna bak (hangi dosyadan geldiğiyle birlikte):

```bash
git config --show-origin --get-all credential.helper
```

macOS'ta çoğunlukla `osxkeychain` çıkar. Git bu durumda bilgiyi Keychain'den almaya
çalışır. Keychain'de `github.com` için geçerli kayıt yoksa (hiç kaydedilmemiş, silinmiş
ya da eski bir şifre kaydı kalmış) soruya düşer.

GitHub CLI kuruluysa onun durumuna da bak:

```bash
gh auth status
```

### Tipik tutarsızlık

`gh` zaten giriş yapmış ve "Git operations protocol: ssh" seçili, ama repo HTTPS ile
klonlanmış. Bu durumda `gh`'nin token'ı **hiç kullanılmaz**, çünkü Git HTTPS için
Keychain'e bakar ve boş döner.

## Çözüm 1: `gh`'yi credential helper yap (HTTPS'te kalmak istiyorsan)

```bash
gh auth setup-git
```

Bu komut `~/.gitconfig` içine `github.com` (ve `gist.github.com`) için şunu yazar:

```ini
[credential "https://github.com"]
    helper =
    helper = !/opt/homebrew/bin/gh auth git-credential
```

Baştaki boş `helper =` satırı, bu host için önceden tanımlı helper'ları (ör.
`osxkeychain`) devre dışı bırakır. Ardından `gh`'nin token'ı kullanılır. Yol, `gh`'nin
kurulu olduğu yere göre değişir.

Doğrulama:

```bash
git config --show-origin --get-all credential.https://github.com.helper
```

## Çözüm 2: Remote'u SSH'a çevir

Kullanıcı adı/token ihtiyacını tamamen ortadan kaldırır.

```bash
git remote set-url origin git@github.com:ORG/REPO.git
```

Bağlantıyı doğrula:

```bash
ssh -T git@github.com
# Hi <kullanıcı>! You've successfully authenticated, but GitHub does not provide shell access.

git ls-remote --heads origin
```

`ssh -T` çıktısındaki "does not provide shell access" bir hata değildir, başarılı
doğrulamanın normal mesajıdır.

SSH anahtarı yoksa önce oluşturup GitHub hesabına eklemek gerekir (`ssh-keygen`, ardından
`gh ssh-key add` ya da GitHub'da Settings → SSH keys).

## Hangisini seçmeli?

- Zaten SSH anahtarın tanımlıysa ve `gh` protokolü `ssh` ise → **Çözüm 2**. Ayarlarınla
  tutarlıdır.
- HTTPS kullanmak zorundaysan (ör. SSH portu kapalı bir ağ) → **Çözüm 1**.
- İkisini birlikte bırakmanın zararı yok. `gh auth setup-git` yalnızca HTTPS remote'ları
  etkiler.

## Hâlâ çalışmıyorsa

- **Kurumsal organizasyon (SAML SSO):** Token ya da SSH anahtarı organizasyon için ayrıca
  yetkilendirilmemiş olabilir. GitHub'da Settings → Applications / SSH keys altında
  "Authorize" (SSO) adımına bak. Token için `gh auth refresh` da gerekebilir.
- **Eski Keychain kaydı:** Bozuk bir kayıt yüzünden hata alıyorsan sil:

  ```bash
  printf 'protocol=https\nhost=github.com\n\n' | git credential-osxkeychain erase
  ```

- **Yanlış hesap:** `gh auth status` ile aktif hesabın doğru olduğunu kontrol et.
