# Aybars

**English** · [Türkçe](#aybars-türkçe)

▶ **[Play in your browser](https://cuneytinann.github.io/Aybars_ChessAI/)**

**A chess bot that plays at about 2,500–2,600 on Stockfish's scale, in a single 7,945-byte HTML file.**

Aybars is [FideLite](https://www.fidelite.art)'s arbiter at the `L2` rule level with a search engine built on top. You play White, or Black in `black.html`, and Aybars answers. Against Stockfish with its strength limited by `UCI_Elo`, it scores like a ~2,520 player at 1 second per move and ~2,585 at 2 seconds. That scale is roughly the CCRL Blitz engine list, not a human FIDE rating. The whole page (board, interface, rules and bot) is 7,945 bytes; zipped, 4,731.

The rules are FideLite's: legal moves, castling, en passant, promotion, check, mate and stalemate, insufficient material, the locked-pawn detector `l` (FIDE 5.2.2), and the repetition and 50-move counters. How each of them is read and applied is documented at **[fidelite.art](https://www.fidelite.art)**. This README covers only what Aybars adds: the bot, and the handful of changes the rule layer needed to carry it.

## At a glance

| | |
|---|---|
| **Base** | FideLite's `L2` rule level (counters and automatic draws; no clock, offers or claims), in the forms of `engine_4x.js`, FideLite's faster build |
| **What's added** | a search (alpha-beta with iterative deepening, a transposition table, quiescence) and an evaluation (material, piece-square tables, pawn structure, mobility, mating a bare king) |
| **Size** | one file, 7,945 bytes; 4,731 bytes zipped. The bot adds 5,719 bytes to FideLite's `L2` (2,226 bytes) |
| **Strength** | ~2,520 at 1 s per move, ~2,585 at 2 s, against limited Stockfish (100 games each, ±35) |
| **Thinking time** | a 2-second budget per move; about 1 second used on average, because the search stops at 12 plies |
| **Locked pawns** | a king-and-pawn structure in which neither side can ever mate ends the game as a draw (`DP`), through FideLite's detector |
| **Playing Black** | `black.html`: the board turned and Aybars moving first; the same file with two values changed |
| **Installation** | none: play online, or download a file and open it |

## Try it now

1. Open [cuneytinann.github.io/Aybars_ChessAI](https://cuneytinann.github.io/Aybars_ChessAI/) to play White, or [black.html](https://cuneytinann.github.io/Aybars_ChessAI/black.html) to play Black. You can also download either file and open it locally.
2. Click one of your pieces, then one of its lit target squares; Aybars answers in a second or two, and in `black.html` it plays the first move. To promote, pick the piece from the buttons that appear above the board.
3. The status line shows `M=` (half-moves counted toward the 50-move rule), `R=` (how many times the current position has occurred) and `C!` when the side to move is in check. When the game ends it shows the result code instead: `W#` White mates, `B#` Black mates, `SM` stalemate, `DP` dead position (insufficient material or locked pawns), `3R` threefold repetition, `50` the 50-move rule. Reload the page for a new game.

Any browser from 2020 on will do; the page needs BigInt (Chrome 67+, Firefox 68+, Safari 14+). On fidelite.art the same file is exhibited as [`L2_aybars_2500`](https://www.fidelite.art/#bots), where **Show code** colours it by layer: rules, interface, bot, and the code that serves both.

## What Aybars changes in the rule layer

Compared definition by definition with FideLite's [`builds/L2/L2.html`](https://github.com/cuneytinann/FideLite/blob/main/builds/L2/L2.html):

| definition | FideLite `L2` | Aybars |
|---|---|---|
| `b` | the board as a plain array | an `Int8Array`, for the bot's speed |
| `G` | the `L2` form | the `engine_4x.js` form; the ray test reads `b[i]<1` instead of `!b[i]`, about 1.5% faster on a typed array |
| `V` | `b.some(…)` over the board | a `while` loop that skips pieces too far from the king to attack it (measured: the loop saves 18.7% of search time, the distance guard about 8%) |
| `L` | yes or no for one target square; copies the board to try the move | the list of a piece's legal targets (the `engine_4x.js` form); tries the move in place and takes it back through the bot's undo stack, leaving the bot's hash and evaluation intact |
| `M` | writes the board directly | writes every square through the bot's `S1`, which keeps the Zobrist hash, the incremental evaluation and the undo stack up to date |
| `l` | up to 600 propagation rounds | the `engine_4x.js` fixed-point version (stops when nothing changes, about 29× faster); `forEach` instead of `map`, since an `Int8Array` cannot hold a BigInt |
| `A` | ends the game on fivefold repetition and 75 moves | ends it on threefold repetition and 50 moves (below); also resets the undo stack and records each position in the bot's game history, so the search sees repetitions from the game itself |
| interface | the board turns to the side to move; coordinates in the status line | your colour always at the bottom; no coordinates; the bot moves after each of your moves |

Everything not in the table is FideLite's, byte for byte, apart from the `indexOf` alias `Q` that both layers share.

**Why threefold repetition and 50 moves.** FideLite's `L2` has no player layer, so the claim thresholds never trigger there and the game ends only at the automatic ones, fivefold and 75 moves. Aybars ends at threefold and 50 on purpose: its search scores a repetition and the 50-move limit as draws, and raising the game's thresholds while the search stays where it is would pull the two apart. FideLite records this as a deliberate exception.

## The bot

Everything after the first `d();` in the script is the bot: 5,510 bytes.

| part | what it does |
|---|---|
| **Move generator** `g` | its own pseudo-legal generator, faster than the rule layer's and checked against it; an illegal move is caught when the reply captures the king. Promotes to queen or knight |
| **Search** `Z` | negamax alpha-beta; a transposition table (32-bit Zobrist hash, 2²¹ slots, kept from move to move); quiescence on captures with stand-pat, delta pruning and SEE pruning of losing captures; reverse futility pruning at depths 1–3 (120 cp per ply); late move reductions for quiet moves from the third one on |
| **Move ordering** `EM` | the hash move first, then captures by victim and attacker (MVV-LVA), the killer move, the counter-move and the history table |
| **Driver** `o` | iterative deepening up to 12 plies within 2 seconds, with a ±40 cp aspiration window from the third iteration; a new iteration starts only if it is likely to finish in time |
| **Evaluation** `j` | material and piece-square tables, kept incrementally; passed pawns (rank² × 4, halved when blocked) and isolated pawns (−25); rooks on open and half-open files; mobility; pieces' closeness to the enemy king; the king's table blended from middlegame to endgame by material; against a bare king, driving it to the edge (to a corner of the bishop's colour with bishop and knight) and bringing the kings together |
| **Draws** | repetitions against the game's history and the search path, and the 50-move limit; a draw counts −20 cp for the bot, so it plays on when it can |

The piece values come from AlphaZero's estimates in Tomašev et al., *Assessing Game Balance with AlphaZero* (pawn 100, knight 305, bishop 333, queen 950), with the rook lowered from 563 to 489. The piece-square tables start from Tomasz Michniewski's Simplified Evaluation Function and were re-fitted by Texel tuning on the Zurichess quiet-labeled positions.

## Strength

Measured against Stockfish 17.1 with its strength limited through `UCI_Elo`, Stockfish thinking 300 ms per move. Limited Stockfish picks its move at a fixed shallow depth, so its strength barely depends on time; Aybars' does. 100 games per row, ±35.

| Aybars' time per move | Stockfish `UCI_Elo` | score | estimate |
|---|---|---|---|
| 0.3 s | 2400 / 2500 | 55.5% / 44% | ~2,445 |
| 1 s | 2500 | 53% | ~2,520 |
| 2 s (as shipped) | 2500 | 62% | ~2,585 |

According to Stockfish's own source, `UCI_Elo` is calibrated to roughly the CCRL Blitz engine list. That is not a human FIDE rating: there is no reliable conversion, and limited Stockfish does not make human mistakes. The honest reading is *about 2,500 at 1 s per move and about 2,580 at 2 s, on limited Stockfish's scale*. The page gives Aybars 2 seconds; it uses about 1 on average, since the search stops at 12 plies.

The `2500` in the name `L2_aybars_2500` is a checkpoint label set before any absolute measurement; the measurement later landed close to it.

## How it was built

Aybars came out of a separate workbench whose one rule was that nothing enters unmeasured. Each change differed from the current base in exactly one place and was played against it in engine matches at 300 ms per move under SPRT (H0 0, H1 +20 Elo, up to 1,500 games). Before any run, gates had to pass: perft 23/23, proof that the candidate differs from the base in one place only, and a check of mate and stalemate scores and of the bot's move generator against the rule layer's. The byte cap was lifted along the way. The code is still written golfed, but a change is judged only by the Elo it brings and the time it costs.

The scope is a bot on top of an arbiter, not a new engine. The rule layer is carried over from FideLite and kept in sync with it, so a change that would alter its form was out of scope, however attractive. This is why alternative board representations, 0x88 among them, were dropped without being measured, and why the base is frozen in its current form.

## Files

| file | size | contents |
|---|---|---|
| `index.html` | 7,945 B | game, rules and bot; you play White |
| `black.html` | 7,944 B | the same with you playing Black: the board turned (`w^7` instead of `w^56`) and the bot's colour `D` set from 0 to 1 |
| `LICENSE` | | MIT License |

Zipped with zopfli (`advzip -z -4`), `index.html` comes to 4,731 bytes and `black.html` to 4,743; with `zip -9`, `index.html` is 4,814.

## License

Aybars is released under the MIT License; see the `LICENSE` file for details.

## Acknowledgments and credits

- **[FideLite](https://github.com/cuneytinann/FideLite):** the rule layer Aybars is built on; its rules and functions are documented at [fidelite.art](https://www.fidelite.art).
- **Nenad Tomašev, Ulrich Paquet, Demis Hassabis, Vladimir Kramnik,** *Assessing Game Balance with AlphaZero: Exploring Alternative Rule Sets in Chess* (2020): the piece values.
- **Tomasz Michniewski,** Simplified Evaluation Function ([chessprogramming.org](https://www.chessprogramming.org/Simplified_Evaluation_Function)): the starting point of the piece-square tables.
- **Alexandru Moșoi, Zurichess:** the `quiet-labeled.epd` positions used for Texel tuning.
- **[Stockfish](https://stockfishchess.org):** the yardstick for the strength measurement.
- **[Chess Programming Wiki](https://www.chessprogramming.org):** the SEE algorithm and the standard search techniques.

---

## Technical appendix

<details>
<summary><b>Bytes per definition</b></summary>

The rule layer, FideLite `L2` against Aybars, without the trailing comma. Definitions not listed are identical.

| definition | FideLite `L2` | Aybars |
|---|---|---|
| `b` | 70 | 81 |
| `Q` | — | 11 |
| `G` | 249 | 255 |
| `V` | 55 | 93 |
| `L` | 90 | 171 |
| `M` | 206 | 218 |
| `l` | 434 | 477 |
| `A` | 243 | 255 |
| `T[N]` | 70 | 79 |
| `d` | 254 | 230 |
| `S` | 88 | 96 |
| **script before the bot** | **1,937** | **2,146** |

The bot, by part:

| part | bytes |
|---|---|
| search `Z` | 1,337 |
| evaluation `j` | 1,140 |
| move generator `g` with direction tables `VD`, `VN` | 663 |
| static exchange `SE` with least-valuable-attacker `LV` | 641 |
| tables: piece values `P`, piece-square `K`, `PS`, phase `PH` | 532 |
| hash and undo: `F`, `H2`, `S1`, `A1`, `A2`, `UN`, `SY` | 393 |
| driver `o` | 282 |
| move ordering `EM` with the move buffer `MV`, `S3` | 248 |
| other state: history, killers, counter-moves, TT, game history, pawn masks | 210 |
| hook that starts the bot after each of your moves | 64 |
| **total** | **5,510** |

Rule-layer changes (+209) and the bot (+5,510) make up the 5,719 bytes over FideLite's `L2`; the HTML around the script is the same 289 bytes in both. The figures are for `index.html`; in `black.html`, `T[N]` is one byte shorter.

</details>

<details>
<summary><b>Measured history</b></summary>

The largest measured steps on the way to the current base. Each row was measured against the base of its day, under the conditions shown; the rows do not add up.

| change | measurement |
|---|---|
| late move reductions fixed (they had been dead code) | +107 ±22, 240 games |
| counter-move history | +79 ±24, SPRT, stopped at game 209 |
| transposition table | +76 ±17, 400 games at 300 ms |
| time use: start a new iteration only if elapsed + 2 × the last one fits the budget (was 4 ×) | +70 ±17, 400 games at 300 ms |
| passed pawns | +65 ±17, 400 games at depth 6 |
| transposition table kept between moves | +44 ±17, 400 games |
| game history fed into the repetition check | +40 ±24 |
| hash move ordered first | +33 ±17, 400 games at 300 ms |
| Texel re-fit of the piece-square tables | +32 ±14, SPRT, stopped at game 620 |
| blocked passed pawns halved, together with draw scores kept out of the transposition table | +31 ±14, combined |
| late move reductions gated on the count of quiet moves (1 / 4 / 9) | +28 ±13, SPRT; measured together with SEE in move ordering, which a later ablation showed added nothing |
| SEE pruning of losing captures in quiescence | +22 ±10, SPRT, stopped at game 1,168 |
| mobility | removing it, even with the tables re-fitted, cost −58 ±24, SPRT |
| make/unmake, `Int8Array`, packed moves, incremental evaluation | ×1.48 nodes per second, play identical |

</details>

---

# Aybars (Türkçe)

[English](#aybars) · **Türkçe**

▶ **[Tarayıcıda hemen oyna](https://cuneytinann.github.io/Aybars_ChessAI/)**

**7.945 baytlık tek bir HTML dosyasında, Stockfish ölçeğinde 2.500–2.600 civarında oynayan bir satranç botu.**

Aybars, [FideLite](https://www.fidelite.art/tr)'ın `L2` kural seviyesindeki hakeminin üstüne kurulmuş bir arama motoru. Sen beyazla, `black.html`'de siyahla oynuyorsun; Aybars cevap veriyor. Gücü `UCI_Elo` ile sınırlanmış Stockfish'e karşı hamle başına 1 saniyede ~2.520'lik, 2 saniyede ~2.585'lik bir oyuncu gibi skor yapıyor. Bu ölçek kabaca CCRL Blitz motor listesi; insan FIDE puanı değil. Sayfanın tamamı (tahta, arayüz, kurallar ve bot) 7.945 bayt; zip'lenince 4.731.

Kurallar FideLite'ın: yasal hamleler, rok, geçerken alma, terfi, şah, mat ve pat, yetersiz materyal, kilitli piyon dedektörü `l` (FIDE 5.2.2), tekrar ve 50 hamle sayaçları. Her birinin nasıl okunup uygulandığı **[fidelite.art](https://www.fidelite.art/tr)**'ta anlatılıyor. Bu README yalnızca Aybars'ın eklediklerini anlatıyor: botu ve kural katmanının onu taşımak için geçirdiği birkaç değişikliği.

## Bir bakışta

| | |
|---|---|
| **Taban** | FideLite'ın `L2` kural seviyesi (sayaçlar ve kendiliğinden gelen beraberlikler; saat, teklif ve talep yok), FideLite'ın hızlı sürümü `engine_4x.js`'in yazımlarıyla |
| **Eklenen** | bir arama (yinelemeli derinleştirmeli alfa-beta, transpozisyon tablosu, sessizlik araması) ve bir değerlendirme (materyal, taş-kare tabloları, piyon yapısı, hareketlilik, yalnız şahı mat etme) |
| **Boyut** | tek dosya, 7.945 bayt; zip'te 4.731 bayt. Bot, FideLite'ın `L2`'sine (2.226 bayt) 5.719 bayt ekliyor |
| **Güç** | sınırlı Stockfish'e karşı hamle başına 1 sn'de ~2.520, 2 sn'de ~2.585 (her biri 100 maç, ±35) |
| **Düşünme süresi** | hamle başına 2 saniyelik bütçe; arama 12 yarım hamlede durduğu için ortalamada yaklaşık 1 saniye kullanılıyor |
| **Kilitli piyonlar** | iki tarafın da hiçbir zaman mat edemeyeceği bir şah-piyon yapısı, FideLite'ın dedektörüyle oyunu beraberlikle (`DP`) bitiriyor |
| **Siyahla oynamak** | `black.html`: tahta çevrilmiş ve ilk hamleyi Aybars yapıyor; iki değeri değiştirilmiş aynı dosya |
| **Kurulum** | yok: çevrimiçi oyna ya da bir dosyayı indirip aç |

## Hemen dene

1. Beyazla oynamak için [cuneytinann.github.io/Aybars_ChessAI](https://cuneytinann.github.io/Aybars_ChessAI/), siyahla oynamak için [black.html](https://cuneytinann.github.io/Aybars_ChessAI/black.html) adresini aç. İki dosyadan birini indirip yerelde de açabilirsin.
2. Kendi taşlarından birine, sonra yanan hedef karelerinden birine tıkla; Aybars bir iki saniye içinde cevap veriyor, `black.html`'de ilk hamleyi de o yapıyor. Terfi için tahtanın üstünde açılan düğmelerden taşı seç.
3. Durum satırında `M=` (50 hamle kuralına sayılan yarım hamleler), `R=` (mevcut pozisyonun kaçıncı kez geldiği) ve sırası gelen taraf şahtayken `C!` görünüyor. Oyun bitince yerine sonuç kodu geliyor: `W#` beyaz mat etti, `B#` siyah mat etti, `SM` pat, `DP` ölü pozisyon (yetersiz materyal ya da kilitli piyonlar), `3R` üçlü tekrar, `50` 50 hamle kuralı. Yeni oyun için sayfayı yenile.

2020 ve sonrasının herhangi bir tarayıcısı yeterli; sayfa BigInt istiyor (Chrome 67+, Firefox 68+, Safari 14+). Aynı dosya fidelite.art'ta [`L2_aybars_2500`](https://www.fidelite.art/tr#bots) adıyla sergileniyor; orada **Kodu göster** onu katmanlarına göre boyuyor: kurallar, arayüz, bot ve ikisine birden hizmet eden kod.

## Aybars'ın kural katmanında değiştirdikleri

FideLite'ın [`builds/L2/L2.html`](https://github.com/cuneytinann/FideLite/blob/main/builds/L2/L2.html) dosyasıyla tanım tanım karşılaştırıldığında:

| tanım | FideLite `L2` | Aybars |
|---|---|---|
| `b` | tahta düz bir dizi | botun hızı için `Int8Array` |
| `G` | `L2` yazımı | `engine_4x.js` yazımı; ışın testi `!b[i]` yerine `b[i]<1`, tipli dizide yaklaşık %1,5 daha hızlı |
| `V` | tahta üzerinde `b.some(…)` | şaha saldıramayacak kadar uzaktaki taşları atlayan bir `while` döngüsü (ölçülen: döngü arama süresinin %18,7'sini, uzaklık bekçisi yaklaşık %8'ini kazandırıyor) |
| `L` | tek bir hedef kare için evet ya da hayır; hamleyi denemek için tahtayı kopyalıyor | bir taşın yasal hedeflerinin listesi (`engine_4x.js` yazımı); hamleyi yerinde deneyip botun geri alma yığınıyla geri alıyor, botun hash'ine ve değerlendirmesine dokunmadan |
| `M` | tahtaya doğrudan yazıyor | her kareyi botun `S1`'i üzerinden yazıyor; Zobrist hash'i, artımlı değerlendirme ve geri alma yığını böylece güncel kalıyor |
| `l` | en fazla 600 yayılım turu | `engine_4x.js`'in sabit noktalı sürümü (değişiklik kalmayınca duruyor, yaklaşık 29 kat hızlı); `Int8Array` BigInt tutamadığı için `map` yerine `forEach` |
| `A` | oyunu beşli tekrar ve 75 hamlede bitiriyor | üçlü tekrar ve 50 hamlede bitiriyor (aşağıda); ayrıca geri alma yığınını sıfırlıyor ve her pozisyonu botun oyun geçmişine yazıyor, böylece arama oyunun kendisinden gelen tekrarları da görüyor |
| arayüz | tahta sırası gelen tarafa dönüyor; durum satırında koordinat var | senin rengin hep altta; koordinat yok; her hamlenden sonra bot oynuyor |

Tabloda olmayan her şey, iki katmanın ortak kullandığı `indexOf` takma adı `Q` dışında, bayt bayt FideLite'ın.

**Neden üçlü tekrar ve 50 hamle.** FideLite'ın `L2`'sinde oyuncu katmanı yok; talep eşikleri orada hiçbir şey tetiklemiyor ve oyun yalnızca kendiliğinden gelen beşli tekrar ve 75 hamlede bitiyor. Aybars bilerek üçlü tekrar ve 50 hamlede bitiyor: araması tekrarı ve 50 hamle sınırını beraberlik sayıyor, oyunun eşiğini yukarı çekip aramayı olduğu yerde bırakmak ikisini birbirinden koparırdı. FideLite bunu bilerek bırakılmış bir istisna olarak kayda geçiriyor.

## Bot

Script'te ilk `d();`'den sonraki her şey bot: 5.510 bayt.

| parça | ne yapıyor |
|---|---|
| **Hamle üretici** `g` | kendi sözde-yasal üreticisi; kural katmanınınkinden hızlı ve ona karşı sınanmış. Kuralsız bir hamle, cevabı şahı aldığında yakalanıyor. Vezire ya da ata terfi ediyor |
| **Arama** `Z` | negamax alfa-beta; transpozisyon tablosu (32 bitlik Zobrist hash'i, 2²¹ yuva, hamleden hamleye korunuyor); alışlar üzerinde sessizlik araması (stand-pat, delta budaması ve kaybettiren alışların SEE ile budanması); 1–3 derinlikte ters futility budaması (yarım hamle başına 120 cp); üçüncü sessiz hamleden itibaren geç hamle indirimi (LMR) |
| **Hamle sıralama** `EM` | önce hash hamlesi, sonra kurban ve saldırana göre alışlar (MVV-LVA), killer hamle, karşı hamle (counter-move) ve geçmiş tablosu |
| **Sürücü** `o` | 2 saniye içinde 12 yarım hamleye kadar yinelemeli derinleştirme; üçüncü turdan itibaren ±40 cp'lik aspirasyon penceresi; yeni tur ancak zamanında bitmesi beklenirse başlıyor |
| **Değerlendirme** `j` | materyal ve taş-kare tabloları, artımlı olarak tutuluyor; geçer piyon (sıra² × 4, önü kapalıysa yarısı) ve izole piyon (−25); açık ve yarı açık hattaki kaleler; hareketlilik; taşların rakip şaha yakınlığı; şahın tablosu materyale göre orta oyundan oyunsonuna harmanlanıyor; yalnız şaha karşı onu kenara (fil ve atla, filin rengindeki köşeye) sürmek ve şahları yaklaştırmak |
| **Beraberlikler** | oyun geçmişine ve arama yoluna karşı tekrarlar, 50 hamle sınırı; beraberlik bot için −20 cp sayılıyor, böylece imkânı varken oynamaya devam ediyor |

Taş değerleri AlphaZero'nun Tomašev ve arkadaşlarının *Assessing Game Balance with AlphaZero* çalışmasındaki kestirimlerinden geliyor (piyon 100, at 305, fil 333, vezir 950); kale 563'ten 489'a indirildi. Taş-kare tabloları Tomasz Michniewski'nin Simplified Evaluation Function tablolarından başlıyor ve Zurichess'in quiet-labeled pozisyonları üzerinde Texel ayarıyla yeniden uyduruldu.

## Güç

`UCI_Elo` ile gücü sınırlanmış Stockfish 17.1'e karşı ölçüldü; Stockfish hamle başına 300 ms düşündü. Sınırlı Stockfish hamlesini sabit ve sığ bir derinlikte seçtiği için gücü süreye neredeyse hiç bağlı değil; Aybars'ınki bağlı. Her satır 100 maç, ±35.

| Aybars'ın hamle başına süresi | Stockfish `UCI_Elo` | skor | tahmin |
|---|---|---|---|
| 0,3 sn | 2400 / 2500 | %55,5 / %44 | ~2.445 |
| 1 sn | 2500 | %53 | ~2.520 |
| 2 sn (sayfadaki ayar) | 2500 | %62 | ~2.585 |

Stockfish'in kendi kaynağına göre `UCI_Elo` kabaca CCRL Blitz motor listesine göre ayarlanmış. Bu bir insan FIDE puanı değil: güvenilir bir çevrim yok, sınırlı Stockfish de insan gibi hata yapmıyor. Dürüst okuma şu: *sınırlı Stockfish'in ölçeğinde hamle başına 1 sn'de yaklaşık 2.500, 2 sn'de yaklaşık 2.580*. Sayfa Aybars'a 2 saniye veriyor; arama 12 yarım hamlede durduğu için ortalamada yaklaşık 1 saniye kullanıyor.

`L2_aybars_2500` adındaki `2500`, hiçbir mutlak ölçüm yapılmadan konmuş bir durak etiketi; ölçüm sonradan ona yakın çıktı.

## Nasıl yapıldı

Aybars, tek kuralı ölçülmemiş hiçbir şeyin girmemesi olan ayrı bir tezgâhtan çıktı. Her değişiklik mevcut tabandan tek bir yerde ayrıldı ve ona karşı, hamle başına 300 ms'lik motor maçlarında SPRT ile oynatıldı (H0 0, H1 +20 Elo, en fazla 1.500 maç). Her koşudan önce kapılar geçmek zorundaydı: perft 23/23, adayın tabandan yalnızca tek bir yerde ayrıldığının kanıtı mat ile pat skorlarının ve botun hamle üreticisinin kural katmanınınkine karşı sınanması. Bayt tavanı yol üstünde kaldırıldı. Kod hâlâ golf üslubuyla yazılıyor, ama bir değişiklik yalnızca getirdiği Elo ve harcattığı zamanla değerlendiriliyor.

Kapsam bir hakemin üstüne bot, yeni bir motor değil. Kural katmanı FideLite'tan taşınıyor ve onunla senkron tutuluyor; dolayısıyla biçimini değiştirecek bir değişiklik, ne kadar cazip olursa olsun, kapsam dışıydı. 0x88 dahil başka tahta temsilleri bu yüzden ölçülmeden bırakıldı; taban da bu yüzden bugünkü hâlinde donduruldu.

## Dosyalar

| dosya | boyut | içerik |
|---|---|---|
| `index.html` | 7.945 B | oyun, kurallar ve bot; sen beyazla oynuyorsun |
| `black.html` | 7.944 B | sen siyahla oynarken aynısı: tahta çevrilmiş (`w^56` yerine `w^7`) ve botun rengi `D` 0'dan 1'e alınmış |
| `LICENSE` | | MIT Lisansı |

Zopfli ile (`advzip -z -4`) zip'lenince `index.html` 4.731, `black.html` 4.743 bayt tutuyor; `zip -9` ile `index.html` 4.814.

## Lisans

Aybars MIT Lisansı ile yayımlanıyor; ayrıntılar `LICENSE` dosyasında.

## Teşekkür ve atıf

- **[FideLite](https://github.com/cuneytinann/FideLite):** Aybars'ın üstüne kurulduğu kural katmanı; kuralları ve fonksiyonları [fidelite.art](https://www.fidelite.art/tr)'ta anlatılıyor.
- **Nenad Tomašev, Ulrich Paquet, Demis Hassabis, Vladimir Kramnik,** *Assessing Game Balance with AlphaZero: Exploring Alternative Rule Sets in Chess* (2020): taş değerleri.
- **Tomasz Michniewski,** Simplified Evaluation Function ([chessprogramming.org](https://www.chessprogramming.org/Simplified_Evaluation_Function)): taş-kare tablolarının başlangıç noktası.
- **Alexandru Moșoi, Zurichess:** Texel ayarında kullanılan `quiet-labeled.epd` pozisyonları.
- **[Stockfish](https://stockfishchess.org):** güç ölçümünün cetveli.
- **[Chess Programming Wiki](https://www.chessprogramming.org):** SEE algoritması ve standart arama teknikleri.

---

## Teknik ek

<details>
<summary><b>Tanım başına bayt</b></summary>

Kural katmanı, FideLite `L2`'ye karşı Aybars, sondaki virgül hariç. Listelenmeyen tanımlar aynı.

| tanım | FideLite `L2` | Aybars |
|---|---|---|
| `b` | 70 | 81 |
| `Q` | — | 11 |
| `G` | 249 | 255 |
| `V` | 55 | 93 |
| `L` | 90 | 171 |
| `M` | 206 | 218 |
| `l` | 434 | 477 |
| `A` | 243 | 255 |
| `T[N]` | 70 | 79 |
| `d` | 254 | 230 |
| `S` | 88 | 96 |
| **bottan önceki script** | **1.937** | **2.146** |

Bot, parça parça:

| parça | bayt |
|---|---|
| arama `Z` | 1.337 |
| değerlendirme `j` | 1.140 |
| hamle üretici `g`, yön tabloları `VD`, `VN` ile | 663 |
| statik alış değerlendirmesi `SE`, en ucuz saldıran `LV` ile | 641 |
| tablolar: taş değerleri `P`, taş-kare `K`, `PS`, evre `PH` | 532 |
| hash ve geri alma: `F`, `H2`, `S1`, `A1`, `A2`, `UN`, `SY` | 393 |
| sürücü `o` | 282 |
| hamle sıralama `EM`, hamle tamponu `MV`, `S3` ile | 248 |
| öteki durum: geçmiş, killer, karşı hamle, TT, oyun geçmişi, piyon maskeleri | 210 |
| her hamlenden sonra botu başlatan kanca | 64 |
| **toplam** | **5.510** |

Kural katmanındaki değişiklikler (+209) ve bot (+5.510), FideLite'ın `L2`'sine eklenen 5.719 baytı oluşturuyor; script'in çevresindeki HTML ikisinde de aynı 289 bayt. Sayılar `index.html` için; `black.html`'de `T[N]` bir bayt kısa.

</details>

<details>
<summary><b>Ölçüm geçmişi</b></summary>

Bugünkü tabana giden yoldaki en büyük ölçülmüş adımlar. Her satır kendi gününün tabanına karşı, gösterilen koşullarda ölçüldü; satırlar toplanmaz.

| değişiklik | ölçüm |
|---|---|
| geç hamle indirimi düzeltildi (ölü koddu) | +107 ±22, 240 maç |
| karşı hamle geçmişi | +79 ±24, SPRT, 209. maçta durdu |
| transpozisyon tablosu | +76 ±17, 300 ms'de 400 maç |
| zaman kullanımı: yeni tur ancak geçen süre + 2 × son tur bütçeye sığarsa (önceden 4 ×) | +70 ±17, 300 ms'de 400 maç |
| geçer piyon | +65 ±17, 6 derinlikte 400 maç |
| transpozisyon tablosunun hamleler arasında korunması | +44 ±17, 400 maç |
| oyun geçmişinin tekrar denetimine verilmesi | +40 ±24 |
| hash hamlesinin öne alınması | +33 ±17, 300 ms'de 400 maç |
| taş-kare tablolarının Texel ile yeniden uydurulması | +32 ±14, SPRT, 620. maçta durdu |
| önü kapalı geçer piyonun yarıya inmesi, beraberlik skorlarının transpozisyon tablosuna yazılmamasıyla birlikte | +31 ±14, birleşik |
| geç hamle indiriminin sessiz hamle sayısına bağlanması (1 / 4 / 9) | +28 ±13, SPRT; hamle sıralamasında SEE ile birlikte ölçüldü, sonraki bir ablasyon SEE'nin katkısının olmadığını gösterdi |
| sessizlik aramasında kaybettiren alışların SEE ile budanması | +22 ±10, SPRT, 1.168. maçta durdu |
| hareketlilik | tablolar yeniden uydurulsa bile çıkarılması −58 ±24'e mal oldu, SPRT |
| yap/geri al, `Int8Array`, paketli hamleler, artımlı değerlendirme | saniyede ×1,48 düğüm, oyun birebir aynı |

</details>
