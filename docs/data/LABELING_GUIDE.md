# Etiketleme rehberi (FAZ 10-A.3)

**Okuyucu:** etiketleyici, inceleyici (reviewer), hakem (adjudicator), İSG uzmanı.
**Kapsam:** `ppe-annotation/1` (DATASET_SPEC.md). Bu rehberdeki tüm örnekler **sentetiktir**; gerçek çalışan
görüntüsü içermez. Gerçek veri ancak DATA_GOVERNANCE.md'deki izin kapısı açıldıktan sonra etiketlenir.

## 1. Temel ilkeler

1. **Gördüğünü etiketle, tahmin etme.** Bölge görünmüyorsa `UNKNOWN`; kişi değerlendirilemeyecek kadar kötü
   görünüyorsa `NOT_EVALUABLE`. Görünmeme ihlal değildir.
2. **Model çıktısı gerçek değildir.** Önetiket (model_prelabel) kullanılırsa her kutu ve durum insan tarafından
   kontrol edilir; YOLO'nun baret bulamaması `MISSING` demek değildir.
3. **İz kimliği kişi kimliği değildir.** `t1`, `t2` yalnız o videoda aynı kişiyi takip etmek içindir. Ad, yüz,
   kıyafet tarifi, çalışan numarası yazılmaz; kişi videolar arasında eşleştirilmez.
4. **Emin değilsen düşük güven ver ve işaretle** (`annotator_confidence: low`) — ikinci göz zorunlu olur.
5. **Kutu:** kişinin görünen tüm gövdesi (kesik ise görünen kısmı); baret kutusu baretin kendisi; yelek kutusu
   görünen yelek alanı.

## 2. Değerlendirilebilirlik eşiği (proje varsayılanı — İSG ile teyit edilecek)

Kişi `NOT_EVALUABLE` sayılır: kişi kutusu yüksekliği **< 40 px** (`too_small`), kişinin yarıdan fazlası kadraj
dışında (`truncated`), ağır bulanıklık (`motion_blur`), karanlık (`low_light`), kalabalıkta gövde ayrılamıyor
(`crowd_overlap`). Bu eşik, üretimdeki güvenilirlik kapısından (60 px, kenar 6 px, güven 0.40) **bağımsızdır**:
GT insan görüşünü ölçer; sistemin kapı yüzünden karar vermediği kişiler (`ANALIZ`) değerlendirmede ayrıca sayılır.

## 3. Durum kartları (A–M)

| # | Durum | Kişi | Baret | Yelek | Not / inceleme |
|---|---|---|---|---|---|
| A | Baret net biçimde mevcut | EVALUABLE | `PRESENT` + baret kutusu | duruma göre | Kutu kişi kutusunun içinde (baş bölgesi) |
| B | Baret net biçimde yok (kafa açıkça görünüyor) | EVALUABLE | `MISSING`, kutu yok | duruma göre | Kritik → ikinci insan zorunlu; şapka/kapüşon/saç = MISSING |
| C | Kafa örtülmüş / görünmüyor (arkası dönük ve kafa kadraj dışı, eşya kafayı kapatıyor) | EVALUABLE, `visibility: head_hidden` | `UNKNOWN` | görünüyorsa PRESENT/MISSING | `MISSING` **yazılamaz** (doğrulayıcı hata verir) |
| D | Baret başka nesnenin arkasında, kısmen görünüyor | EVALUABLE, `occlusion: partial` | görünen kısım baret olduğu açıksa `PRESENT` + görünen kısım kutusu; değilse `UNKNOWN` | — | Emin değilsen `confidence: low` |
| E | Kişi kadraj kenarında | yarıdan fazlası dışarıdaysa `NOT_EVALUABLE (truncated)`; değilse EVALUABLE + `truncated: true` | NOT_EVALUABLE ya da bölge görünüyorsa PRESENT/MISSING | aynı | Üretimde kenar kapısı bu kişiyi `ANALIZ`'de bırakır — GT bunu ayrıca ölçer |
| F | Uzak ve küçük çalışan | < 40 px → `NOT_EVALUABLE (too_small)`; ≥ 40 px ise EVALUABLE | eşik üstünde baret seçilemiyorsa `UNKNOWN` | aynı | Uzak çalışan senaryosu ayrı etiketlenir (`far_workers`) |
| G | Yelek kısmen görünür | EVALUABLE | — | yelek olduğu açıksa `PRESENT` + görünen kısım; gövde görünmüyorsa `UNKNOWN` (`torso_hidden`) | Benzer renkli kıyafet = yelek değildir (şerit/reflektör yoksa `MISSING` ancak gövde açıkça görünüyorsa) |
| H | İş makinesi insana benziyor | **kişi etiketlenmez** | — | — | Model kişi dese bile GT'de kişi yok; makine kolu, manken, levha da öyle. Model FP'si ölçümde görünür |
| I | Kalabalık, kişiler örtüşüyor | her kişi ayrı; ayrılamıyorsa `NOT_EVALUABLE (crowd_overlap)` | ekipmanı **doğru kişiye** bağla | aynı | Ekipman başkasının kutusuna düşüyorsa doğrulayıcı uyarır; kritik |
| J | Düşük ışık | ekipman seçilebiliyorsa EVALUABLE; değilse `NOT_EVALUABLE (low_light)` | seçilemiyorsa `UNKNOWN` | aynı | Senaryo etiketi `low_light` |
| K | Hareket bulanıklığı | aynı mantık, `motion_blur` | | | Tek kare bulanıksa komşu karelere bakarak karar verme — o karede göreni yaz |
| L | Track ID değişimi (aynı kişi) | aynı kişi kesintisiz görünüyorsa **aynı** `track_id` | — | — | GT kimliği modelin ID'sinden bağımsızdır; modelin ID değişimi ölçümde "ID switch" olarak görünür |
| M | Kişi kadrajdan çıkıp geri geliyor | aynı kişi olduğundan **eminsen** aynı `track_id` + dönüşte `reentry: true`; emin değilsen yeni `track_id` | — | — | İşaretsiz dönüş doğrulayıcıda hata (`E_TRACK_REENTRY`); kimlik tahmini için yüz/ad kullanılmaz |

## 4. Incident (olay) etiketleme

- Aynı kişinin aynı ekipman eksikliğinin **kesintisiz** aralığı = bir incident. Başlangıç: eksikliğin ilk açıkça
  görüldüğü an; bitiş: ekipmanın yeniden görüldüğü an veya kişinin kadrajdan çıktığı an; video sonunda sürüyorsa
  `end_sec: null`.
- Kafa birkaç saniye görünmeyip (UNKNOWN) ardından yine baretsiz görünüyorsa incident **bölünmez**; `uncertainty:
  uncertain` yazılabilir.
- Her incident ikinci insan incelemesinden geçer.

## 5. Yanlış etiketin metriklere etkisi

| Hatalı etiket | Etkisi |
|---|---|
| Görünmeyen kafaya `MISSING` | Sistem doğru şekilde susarsa **sahte FN** → recall olduğundan düşük; sistem alarm verirse sahte TP → recall olduğundan yüksek |
| Görünmeyen kafaya `PRESENT` | Sistem alarm verirse **sahte FP** → yanlış alarm oranı şişer |
| Model bulamadı diye `MISSING` (önetiketi körü körüne kabul) | Modelin kendi hatası GT'ye girer; FN'ler görünmez olur → **recall şişer** (en tehlikeli hata) |
| Makineyi kişi etiketlemek (H) | Sistemin makine FP'si TP sayılır → yanlış alarm gizlenir |
| Küçük/uzak kişiyi `NOT_EVALUABLE` yerine `MISSING` | Kapı nedeniyle susan sistem FN sayılır (bu doğru bir risktir ama eşikle tutarsız etiket ölçümü bozar) |
| Ekipmanı yanlış kişiye bağlamak (I) | İlişki doğruluğu ve kişi-kare TP/FP yer değiştirir |
| Aynı kişiye iki `track_id` (L/M) | Model ID'si doğruyken "parçalanma" görünür; duplicate incident sayısı yanlış |
| Incident sınırını kaydırmak | Başlangıç gecikmesi ve "kaçırılan/yanlış incident" sayısı bozulur |

## 6. Kalite güvencesi ve uyum (IAA)

**İki aşama:** (1) etiketleyici, (2) bağımsız inceleyici (etiketleyiciden farklı kişi). Kritik etiketler (MISSING,
UNKNOWN, NOT_EVALUABLE, düşük güven, önetiket, her incident) için ikinci aşama **zorunlu**; anlaşmazlıkta hakem
(adjudicator) yazılı çözüm verir. Otomatik çoğunluk oyu kullanılmaz.

**Uyum ölçümü (her yeni etiketleyici ve her veri kümesi sürümünde):**
1. Test kümesi dahil her split'ten rastgele (seed kayıtlı) **≥ %10** medya, iki bağımsız etiketleyiciye **körlemesine**
   etiketletilir.
2. `python -m evaluation data-agreement A.json B.json` → kutu uyumu ve durum uyumu **ayrı** raporlanır:
   - **Kutu:** IoU ≥ 0.5 ile eşleşen kişi oranı (F1), ortalama/en düşük IoU, yalnız birinde olan kişiler.
   - **Durum (baret, yelek ayrı):** ham uyum, Cohen kappa (4 kategori), karışıklık tablosu, özellikle
     MISSING anlaşmazlığı sayısı.
3. Tüm anlaşmazlıklar hakeme gider; çözüm belgeye `adjudicated` + `resolution` olarak yazılır.
4. **Kabul eşikleri** (ör. kappa alt sınırı) İSG ve veri sorumlusuyla belirlenir; belirlenene kadar IAA sonucu
   "ölçüldü" olarak raporlanır, PASS denmez. Teknik öneri (onay değil): MISSING anlaşmazlıkları sıfıra indirilmeden
   test kümesi dondurulmaz.

## 7. Sentetik örnek (gerçek veri değildir)

```json
{"track_id": "t1", "box": [100, 100, 220, 460], "visibility": "head_hidden", "occlusion": "partial",
 "truncated": false, "annotator_confidence": "high", "person_status": "EVALUABLE", "label_origin": "human",
 "helmet": {"status": "UNKNOWN", "box": null},
 "vest": {"status": "PRESENT", "box": [110, 200, 210, 340]},
 "review": {"status": "agreed", "reviewer": "rev_B"}}
```
Tam sentetik veri kümesi ve tüm hata senaryoları: `python -m evaluation data-fixtures --out <dizin> --scenario <ad>`
(senaryolar: `evaluation/fixtures.py`).
