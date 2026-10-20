# Claude Code ve Codex için Harness eklentisi

![Harness](assets/logo.svg)

**Diller:** [English](README.md) · [Deutsch](README.DE.md) · **Türkçe**

Harness, bir kodlama agent'ının talimatlarında (`CLAUDE.md`, `AGENTS.md` ve Skill'ler)
yapılan bir değişikliğin, Claude Code'un veya Codex'in bir projenin daha önce başarısız
olduğu kendi kontrollerini güvenilir biçimde geçmesini sağlayıp sağlamadığını ölçer.
İncelenmiş bir adayı özetini (digest) yazarak benimseyene kadar projenizde hiçbir şey
değişmez ve her benimseme geri alınabilir.

## Neden Harness

Model sabittir: Claude'u veya GPT'yi yeniden eğitemezsiniz. Kontrol ettiğiniz şey onu
çevreleyen sistemdir, yani agent'ınızın her oturumda yüklediği talimatlar ve Skill'ler.
Çoğu ekip bu sistemi sezgiyle değiştirir: bir kural ekler, bir iki kez dener, işe
yaramış gibi görünürse tutar. Bu kanıt anekdottur. Temel çizgisi olmayan tek bir
örnektir, CLI arada güncellenmiş olabilir ve kişisel hook'lar, MCP sunucuları ve
eklentiler sonucu etkiler. Hiçbir işe yaramayan kurallar kalır ve her satır her oturumda
yüklenir.

Harness böyle bir değişikliği kontrollü bir deneye dönüştürür. Değiştirilmemiş temel
çizgi ve bir aday, aynı sabitlenmiş ortamda aynı görevi tekrar tekrar çalıştırır ve her
çalıştırmanın ardından aynı deterministik kontroller gelir. Yanıt ya incelemeniz için
tam bir diff içeren kanıtlanmış bir iyileştirmedir ya da **No demonstrated improvement**
(kanıtlanmış iyileştirme yok).

### Harness ile ve Harness olmadan

| | Harness olmadan | Harness ile |
| --- | --- | --- |
| Kanıt | Bir iki elle deneme | Temel çizgiye karşı aday, tekrarlı, aynı kontroller |
| Model rastlantısallığı | Fark edilmez ("bu sefer çalıştı") | Tekrarlar uyuşmalıdır, aksi hâlde iddia yok |
| Ortam | Günlük kurulumunuz; CLI sürümü, hook'lar ve MCP sunucuları değişir | Sabitlenmiş CLI, boş yapılandırma, ağ yok |
| Güvenlik | Agent gerçek projenizde çalışır | Sandbox içinde özel bir kopya; benimseyene kadar projenize dokunulmaz |
| Model kullanımı | Sınırsız ve izlenmez | İstek, oturum, token ve süre için katı sınırlar |
| Hiçbir şey işe yaramazsa | Talimat dosyası yine de büyür | Hiçbir şey değişmez ve sonuç bunu söyler |
| Geri alma | Hatırlanırsa `git revert` | Gösterilen diff'e bağlı `harness rollback` |
| Geriye kalan | Etkisi bilinmeyen bir kural | Kanıtı olan bir kural ve onu tanımlayan test |

### Harness ne zaman değer

- **Değer:** bir testin yakalayabildiği tekrarlayan agent hataları için, bir ekibin
  paylaştığı talimat dosyaları için, uyulması gereken kurallar için ve haftada çok
  sayıda oturum bir çalıştırmanın maliyetini amorti ettiğinde.
- **Değmez:** agent görevi zaten çözüyorsa (iyileşecek bir şey yoktur, sonuç
  *No demonstrated improvement* olur), beklentiyi hiçbir deterministik kontrol ifade
  edemiyorsa, hata talimatlardan değil model yeteneğinden kaynaklanıyorsa veya tek
  seferlik görevlerde.

## Nasıl çalışır

```text
You: project + task + approved files + acceptance checks + limits
                           |
        Preflight: sandbox, trusted host CLI, limits, credential, checks
                           |
   Core: baseline -> Propose -> Probe -> Compose -> Select
                           |
   Native sessions (Claude Code or Codex CLI) in Bubblewrap, no network
                 |                                   |
     loopback forwarder  -->  provider gateway (holds the credential,
                              caps requests, meters tokens and cost)
                           |
   Independent validation: fresh baseline and candidate rollouts + approved checks
                           |
   Eligibility: improved / unchanged / regressed / indeterminate
                           |
   You review the exact diff -> explicit adoption -> safe rollback
```

1. **Ön kontrol (preflight)**, çalışan bir sandbox, güvenilir bir ana makine CLI'ı,
   eksiksiz sınırlar, bir kimlik bilgisi ve sandbox içinde çalışabilen kontroller
   olmadan başlamayı reddeder.
2. **Çekirdek.** Propose–Probe–Compose çekirdeği değiştirilmemiş temel çizgiyi
   çalıştırır, ana makinenin talimat düzenlemeleri önermesini sağlar, bunları dener ve
   birleştirir ve son bir aday seçer. `k=1` olduğunda bu seçim önemsizdir.
3. **Bağımsız doğrulama**, temel çizginin ve adayın taze, dönüşümlü çalıştırmalarını
   yürütür; her birinin ardından onaylı kontroller ağa kapalı bir sandbox içinde çalışır.
4. **Uygunluk.** Bir aday yalnızca onaylı her kontrol onun için geçtiğinde, temel
   çizginin en az bir başarısızlığı giderildiğinde, hiçbir şey gerilemediğinde ve
   tekrarlar uyuştuğunda uygundur. Aksi hâlde Harness **No demonstrated improvement**
   bildirir. Seçim hiçbir zaman iyileştirmenin kanıtı değildir.
5. **Benimseme** yalnızca size aittir: tam diff'i incelersiniz ve özetini (digest)
   terminalinize yazarsınız. Geri alma da aynı türden bir özete bağlıdır.

Ne model ne de bir eklenti Skill'i harcamaya yetki verebilir, kapsamı genişletebilir
veya canlı projenizi değiştirebilir. Desteklenen talimat yüzeyleri, listelediğiniz
dosyalarla sınırlı olarak: Claude Code için `CLAUDE.md` ve
`.claude/skills/<name>/SKILL.md`; Codex için `AGENTS.md` ve
`.agents/skills/<name>/SKILL.md`.

## Eklentinin içeriği

Üç Skill, bir başvuru belgesi, üç dilde bu README ve OpenAI eklenti dizini kaydı için
iki simge. Skill'ler ve başvuru belgesi İngilizcedir. Hook, MCP sunucusu, komut, agent
veya çalıştırılabilir dosya yoktur.

- `setup`: kurulu çalışma zamanını denetler, doctor'ını ve çevrimdışı bir demoyu
  çalıştırır ve bir proje yapılandırmasının önizlemesini gösterir.
- `refine`: bir iyileştirmeyi model çağrısı yapmayan bir deneme çalıştırmasıyla (dry run)
  önizler, bir iyileştirmeyi ancak harcamaya izin verdikten sonra çalıştırır veya mevcut
  bir adayı değerlendirir.
- `review`: bir çalıştırmanın kanıtlarını ve adayın tam diff'ini gösterir, uygun olup
  olmadığını açıklar ve onaylı bir benimseme, geri alma veya kurtarmada yol gösterir.
- `references/runtime-setup.md`: çalışma zamanının nasıl kurulacağı, doğrulanacağı,
  güncelleneceği ve kaldırılacağı.

## Ne çalıştırır, ne gönderir, ne indirir

Skill'ler, asistanın ayrıca kurulan `harness` komut satırı programını (Python 3.12,
Apache-2.0) bilgisayarınızda çalıştırmasını ister, örneğin:

```bash
harness --version
harness doctor --json
harness demo --mode parallel --k 1 --json
harness init --project PROJECT_PATH --json
```

- Eklentinin kendisi kod içermez, veri göndermez ve hiçbir hizmetle iletişim kurmaz.
- Çalışma zamanının kurulumu, `harness-1.0.0-py3-none-any.whl` ve `SHA256SUMS`
  dosyalarını https://github.com/devsolux/harness-plugin deposunun `v1.0.0` sürümünden
  indirir. Setup Skill'i sağlama toplamını doğrular ve herhangi bir şey kurmadan önce
  sorar.
- `harness refine --allow-model-calls`, seçilen ana makine CLI'ını (Claude Code veya
  Codex) sandbox içinde onaylı proje dosyalarının bir kopyası üzerinde çalıştırır. Bu
  CLI görevi ve okuduklarını yerel bir Harness ağ geçidi üzerinden sağlayıcısına
  (Anthropic veya OpenAI) gönderir; ağ geçidi kimlik bilgisini sandbox dışında tutar ve
  istek, token ve maliyet sınırlarını uygular. Bu seçenek olmadan Harness hiçbir model
  çağrısı yapmaz.
- `harness login claude` ve `harness login codex`, sağlayıcının kendi oturum açma
  işlemini terminalinizde başlatır ve elde edilen token'ı ağ geçidi için saklar.
- macOS'ta veya Bubblewrap olmayan Linux'ta `harness-docker build` yerel bir Linux imajı
  oluşturur. Bunun için digest ile sabitlenmiş bir Ubuntu 24.04 temel imajını, Ubuntu
  paketlerini ve sabitlenmiş Claude Code ve Codex derlemelerini indirir. İmaj hiçbir
  zaman yüklenmez.
- Çalıştırmalar, kanıtlar ve yedekler yerel bir durum dizininde kalır. Harness'in
  telemetrisi yoktur; tek ağ istemcisi, ana makine CLI'ının isteklerini ileten ağ
  geçididir.

## Gereksinimler

- Python 3.12 ve `uv` gibi yalıtılmış bir araç yükleyici.
- Sandbox içinde değerlendirme ve iyileştirme için Bubblewrap'li, root olmayan Linux ya
  da macOS'ta Linux çalışma alanı imajı için Docker.
- Model destekli iyileştirme için: sistem genelinde kurulu Claude Code veya Codex ve
  kendi Claude.ai ya da ChatGPT oturumunuz veya dışa aktardığınız bir API anahtarı.

## Durum

Sürüm 1.0.0. Çevrimdışı çalışma zamanı, loopback bir sağlayıcıya karşı her iki gerçek
ana makine CLI'ı ve benimseme ile geri alma dahil canlı bir Claude Code iyileştirmesi
doğrulandı. CI dışında desteklenen, root olmayan bir Linux ana makinesinde kabul testi
hâlâ bekliyor; bu nedenle gerçek çalıştırmaları gözetim altında yapın.

## Lisans

Apache-2.0. Tam metin `LICENSE` dosyasındadır.
