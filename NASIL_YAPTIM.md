# Maaş Tahmini Modelini Nasıl Geliştirdim

Veri setinde 6704 kişi var. Her biri için yaş, cinsiyet, eğitim, meslek, deneyim yılı ve maaş bilgisi bulunuyor. Amaç maaşı tahmin etmek.

Derste yaptığımız notebook'ta (`class.ipynb`) sırasıyla şunlar vardı:

1. Eksik değerli satırları sildik, 6698 satır kaldı.
2. Cinsiyet, eğitim ve meslek sütunlarını LabelEncoder ile sayıya çevirdik.
3. Korelasyon tablosuna ve grafiklere baktık.
4. IQR yöntemiyle aykırı maaşları sildik.
5. Deneyimle çok ilişkili olduğu için yaşı çıkardık.
6. Veriyi %80 eğitim, %20 test diye ayırıp Linear Regression kurduk.

Bu modelin sonucu **R² = 0.666**, ortalama hata da yaklaşık **24.448 $** idi.

## Fark ettiğim sorunlar

Önce modeli değiştirmek yerine veriye daha dikkatli baktım. Birkaç şey dikkatimi çekti:

**Eğitim sütunu tutarsız yazılmış.** Aynı derece hem `Bachelor's` hem `Bachelor's Degree` olarak geçiyor. Master için de durum aynı, doktorada ise bir yerde `phD` yazıyor. Model bunları farklı eğitim seviyeleri sanıyordu.

**Meslekleri sayıya çevirme şeklimiz yanlıştı.** LabelEncoder 190 küsur mesleğe alfabetik sırayla 0, 1, 2… diye numara veriyor. Linear Regression bu sayıları gerçek bir büyüklük gibi kullanıyor, yani "142 numaralı meslek 18 numaralıdan daha değerli" gibi anlamsız bir şey öğreniyor. Korelasyon tablosunda meslek–maaş ilişkisi 0.06 çıkmıştı. Bunun sebebi mesleğin önemsiz olması değil, yanlış kodlanması.

**IQR adımı aslında hiçbir satırı silmiyormuş.** Alt sınır negatif çıktığı için 350 $, 500 $ gibi açıkça hatalı maaşlar veride kalmış. Bunu notebook'ta silinen satır sayısını yazdırarak kontrol ettim, sonuç 0.

**Model doğrusal.** Mesela kıdemli bir veri bilimcinin maaşı deneyimle bir şekilde artarken, bir satış elemanınınki çok daha farklı artıyor. Düz bir doğru bu tür ilişkileri yakalayamıyor.

## Neleri değiştirdim

- Eğitim etiketlerini birleştirip dört seviyeye indirdim ve sıralı kodladım: Lise = 0, Lisans = 1, Yüksek Lisans = 2, Doktora = 3.
- Meslek isimlerindeki yazım hatasını düzelttim (`Juniour` → `Junior`).
- Meslek adından kıdem bilgisini ayrı sütunlar olarak çıkardım. İsminde "Senior", "Junior" ya da "Manager/Director/Head" geçip geçmediğine baktım.
- Meslekleri LabelEncoder yerine **TargetEncoder** ile kodladım. Bu yöntemde her meslek, eğitim verisindeki ortalama maaşıyla temsil ediliyor. Böylece meslek gerçekten anlamlı bir sayıya dönüşüyor. scikit-learn bunu kendi içinde çapraz doğrulamayla yaptığı için test verisinden bilgi sızmıyor.
- Mesleği ayrıca **One-Hot** ile de kodladım (her meslek için ayrı bir 0/1 sütunu). Böylece model hem "bu meslek ortalama ne kazanıyor" bilgisini hem de mesleğin kendisini görüyor.
- Yaşı geri ekledim. Ağaç tabanlı modeller birbiriyle ilişkili sütunlardan etkilenmiyor ve yaş biraz ek bilgi veriyor.
- **İşe başlama yaşı** (`Start_Age` = yaş − deneyim) diye yeni bir sütun ekledim.
- 350 $, 500 $ gibi açıkça hatalı maaşları (10.000 $ altı) **sadece eğitim verisinden** çıkardım. Test verisine dokunmadım, yani karşılaştırma hâlâ adil.
- Model olarak **ExtraTrees** kullandım. Random Forest gibi çok sayıda karar ağacının ortalamasını alıyor; farkı, ağaçlardaki bölme noktalarını rastgele seçmesi. Bu onu kopya satırları ezberlemeye karşı biraz daha dayanıklı yapıyor. Random Forest, Gradient Boosting ve ikisinin ortalamasını da denedim; nasıl seçtiğimi aşağıda anlatıyorum.

## Sonuçlar

Karşılaştırmanın adil olması için derste kullandığımız bölmeyi aynen kullandım (`test_size=0.2, random_state=42`). Yani iki model de aynı test satırları üzerinde ölçüldü.

| Model | R² | Ortalama hata |
|---|---|---|
| Dersteki Linear Regression | 0.666 | 24.448 $ |
| Aynı Linear Regression, sadece veriyi düzelttikten sonra | 0.839 | 15.347 $ |
| **ExtraTrees (son model)** | **0.977** | **2.760 $** |

Ortalama hata %89 azaldı. Burada benim için en öğretici kısım ikinci satır oldu. Modeli hiç değiştirmeden, sadece veriyi düzgün hazırlayınca skor 0.67'den 0.84'e çıktı. Kalan artış ise doğrusal olmayan modele geçmekten geldi.

## Kendi sonucumu nasıl kontrol ettim

Çalışırken veride çok sayıda tekrar eden satır olduğunu fark ettim: 6698 satırın sadece 1787'si birbirinden farklı. Rastgele bölünce aynı kişi hem eğitim hem test verisine düşebiliyor. Bu durumda model aslında tahmin etmiyor, ezberliyor olabilir. Skorun bu yüzden şişmiş olabileceğinden şüphelendim.

Bunu test etmek için **GroupKFold** kullandım. Bu yöntem aynı satırın bütün kopyalarını aynı tarafa koyuyor. Böylece model test ettiği satırı eğitimde hiç görmemiş oluyor.

| Model | GroupKFold R² |
|---|---|
| Dersteki Linear Regression | 0.656 |
| Son model | 0.919 |

Bu daha zor testte skor 0.98'den 0.92'ye düşüyor, yani tekrar eden satırlar gerçekten biraz etki ediyormuş. Ama fark hâlâ çok büyük, bu yüzden iyileşmenin gerçek olduğunu düşünüyorum.

## Modeli nasıl seçtim

Test verisine bakarak model seçersem test artık "hiç görülmemiş" veri olmaz ve skor iyimser çıkar. Bu yüzden adayları sadece eğitim verisinde, 5 katlı çapraz doğrulamayla (yine kopyaları aynı tarafta tutan GroupKFold ile) karşılaştırdım. Notebook'ta teste sadece en sonda, seçilen model için bir kez bakılıyor. Açık olmak gerekirse, çalışma sırasındaki denemelerde bazı modellerin test skoruna da bakmıştım; son seçimi ise bu şekilde, sadece eğitim verisine dayanarak yaptım.

| Aday | Eğitim verisinde CV R² |
|---|---|
| Gradient Boosting | 0.901 |
| Random Forest | 0.906 |
| Random Forest + Gradient Boosting | 0.911 |
| **ExtraTrees** | **0.923** |

ExtraTrees en iyi çıktığı için onu seçtim.

## Çalıştırmak için

Bütün adımlar, çıktılarıyla birlikte `improved_model.ipynb` dosyasında. Derste kullandığımız kütüphaneler dışında ek bir şey gerekmiyor.
