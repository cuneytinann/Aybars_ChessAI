# Aybars

**English** · [Türkçe](#aybars-türkçe)

▶ **[Play in your browser](https://cuneytinann.github.io/Aybars_ChessAI/)**

**A chess bot on top of FideLite's rules, in a single 8,269-byte HTML file.**

Aybars is a search engine built on [FideLite](https://www.fidelite.art)'s arbiter at the `L2` rule level. You play White, or Black in `black.html`, and Aybars answers. The whole page (board, interface, rules and bot) is 8,269 bytes; zipped, 4,972. An earlier build scored like a ~2,585 player at 2 seconds per move against Stockfish limited by `UCI_Elo`, a scale close to the CCRL Blitz engine list rather than a human FIDE rating; the current build beats that one in direct matches ([Strength](#strength)).

## At a glance

| | |
|---|---|
| **Rules** | FideLite's `L2` rule core, byte for byte, on the 0x88 board |
| **Bot** | a search (alpha-beta with iterative deepening, a transposition table, quiescence) and an evaluation (material, piece-square tables, pawn structure, mobility, mating a bare king) |
| **Size** | one file, 8,269 bytes; 4,972 bytes zipped. FideLite's `L2.html` is 1,922 bytes; the bot itself is 6,227 |
| **Thinking time** | a 2-second budget per move, about 1.8 s used on average; up to 32 plies |
| **Game endings** | mate, stalemate, dead position (insufficient material or locked pawns), threefold repetition, the 50-move rule |
| **Playing Black** | `black.html`: the board turned and Aybars moving first; the same file with two values changed |
| **Installation** | none: play online, or download a file and open it |

## Try it now

1. Open [cuneytinann.github.io/Aybars_ChessAI](https://cuneytinann.github.io/Aybars_ChessAI/) to play White, or [black.html](https://cuneytinann.github.io/Aybars_ChessAI/black.html) to play Black. You can also download either file and open it locally.
2. Click one of your pieces, then one of its lit target squares; Aybars answers in a second or two, and in `black.html` it plays the first move. To promote, pick the piece from the buttons that appear above the board.
3. The status line shows `M=` (half-moves counted toward the 50-move rule), `R=` (how many times the current position has occurred) and `C!` when the side to move is in check. When the game ends it shows the result code instead: `W#` White mates, `B#` Black mates, `SM` stalemate, `DP` dead position (insufficient material or locked pawns), `3R` threefold repetition, `50` the 50-move rule. Reload the page for a new game.

Any browser from 2020 on will do; the page needs BigInt (Chrome 67+, Firefox 68+, Safari 14+). On fidelite.art the same file is exhibited as [`L2_aybars_2500`](https://www.fidelite.art/#bots), where **Show code** colours it by layer: rules, interface and bot.

## The rules: FideLite

Aybars does not write rules of its own. Its rule layer is FideLite's `L2` level, [`builds/L2/L2.html`](https://github.com/cuneytinann/FideLite/blob/main/builds/L2/L2.html), copied verbatim: the five functions that make up the core are identical byte for byte — `C` (the castling rights a square touches), `G` (whether a piece can make a move), `T` (whether a square is attacked), `U` (making a move) and `L` (legality). They give legal moves, castling, en passant, promotion, check, mate and stalemate; on top of them sit the dead-position detector (insufficient material and locked pawns, FIDE 5.2.2) and the repetition and 50-move counters. How each rule is read and applied is documented at **[fidelite.art](https://www.fidelite.art/#rules)**; the rule levels themselves under [`L1` and `L2`](https://www.fidelite.art/#levels).

The rule layer plays and judges the moves made on the board, once per move; it is not on the search path. Where Aybars differs from `L2`:

| | FideLite `L2` | Aybars |
|---|---|---|
| game ends on | fivefold repetition and 75 moves, automatically | threefold repetition and 50 moves (below) |
| repetition table | `R=G[…]=1`: the function `G` doubles as the table, the count in `R` | `R={[…]:$=1}`, the count in `$` |
| dead-position detector | inline in `A`, with the board scan in the driver | the same logic wrapped as `l()`, its variables local, so that its names cannot clash with the bot's |
| additions | — | `a=Math.abs`, used by the bot; the interface's target square `f` in the opening array |
| interface | the board turns to the side to move; coordinates in the status line | your colour always at the bottom; no coordinates; the bot moves after each of your moves |

**Why threefold repetition and 50 moves.** FideLite's `L2` has no player layer, so the claim thresholds never trigger there and the game ends only at the automatic ones, fivefold and 75 moves. Aybars ends at threefold and 50 on purpose: its search scores a repetition and the 50-move limit as draws, and raising the game's thresholds while the search stays where it is would pull the two apart. FideLite records this as a deliberate exception.

## The bot

Everything from `U1=0` to the end of the script is the bot, plus a 22-byte hook inside `A` that hands each move to it: 6,227 bytes in all. The bot shares no code with the rule layer. It is free to chase speed, so it keeps its own copy of the board, its own move generator and check test, and its own make and unmake.

| part | what it does |
|---|---|
| **Board** | at the start of each search the bot takes a typed copy of the board (an `Int8Array` of 128 cells, off-board cells = 7) and hands the board back at the end. Every square it writes goes through `S1`, which keeps the Zobrist hash, the incremental evaluation, the game phase, the king squares and the undo log up to date |
| **Move generator** `g` | pseudo-legal, with castling (the right, the empty squares, the king's square and the one it crosses not attacked) and the legality of en passant (`EPL`) built in. An illegal move is caught at the top of the child node: if the side to move could capture the king, the node returns at once. Promotes to queen or knight |
| **Check test** `V` | rays out from the king's square: eight directions and eight knight jumps, with pawns and kings tested on the first square of a ray |
| **Search** `Z` | negamax alpha-beta; a transposition table (32-bit Zobrist hash, 2²¹ entries of three numbers in one `Int32Array`, kept from move to move); quiescence on captures and promotion pushes, with stand-pat, delta pruning and SEE pruning of losing captures; reverse futility pruning at depths 1–3 (120 cp per ply, not in check); late move reductions for quiet moves: one ply from the third, two from the sixth above depth 4, three from the eleventh above depth 6 |
| **Move ordering** `EM` | the hash move first, then captures by victim and attacker (MVV-LVA), promotions, the killer move, the counter-move and the history table |
| **Driver** `o` | iterative deepening up to 32 plies within 2 seconds, with a ±40 cp aspiration window from the third iteration; a new iteration starts only if the time spent plus twice the last iteration fits the budget |
| **Evaluation** `j` | material and piece-square tables, kept incrementally, with a cache of 2¹⁸ results; passed pawns (rank² × 4, halved when blocked) and isolated pawns (−25); rooks on open and half-open files; mobility; pieces' closeness to the enemy king; the king's table blended from middlegame to endgame by material; once one side has nothing but its king, driving that king to the edge (to a corner of the bishop's colour with bishop and knight) and bringing the kings together |
| **Draws** | repetitions against the game's history and the search path, and the 50-move limit; a draw counts −20 cp for the bot, so it plays on when it can |

There is no clock check inside the search, so an iteration that runs longer than predicted takes the move over the budget: on a slow machine, 4 of 28 test positions took more than 2 seconds, the longest 7.

The piece values come from AlphaZero's estimates in Tomašev et al., *Assessing Game Balance with AlphaZero* (pawn 100, knight 305, bishop 333, queen 950), with the rook lowered from 563 to 489. The piece-square tables start from Tomasz Michniewski's Simplified Evaluation Function and were re-fitted by Texel tuning on the Zurichess quiet-labeled positions.

## Strength

Aybars was measured against Stockfish 17.1 with its strength limited through `UCI_Elo`, Stockfish thinking 300 ms per move. Limited Stockfish picks its move at a fixed shallow depth, so its strength barely depends on time; Aybars' does. The measurement was made on an earlier build: the same search and evaluation on a 64-square board, its depth capped at 12 plies. 100 games per row, ±35.

| earlier build's time per move | Stockfish `UCI_Elo` | score | estimate |
|---|---|---|---|
| 0.3 s | 2400 / 2500 | 55.5% / 44% | ~2,445 |
| 1 s | 2500 | 53% | ~2,520 |
| 2 s | 2500 | 62% | ~2,585 |

The current build has not been measured against Stockfish again. Against the earlier build:

| match | conditions | result |
|---|---|---|
| the 0x88 board against the 64-square board, both capped at 12 plies | 200 ms per move, 100 games | +49 =26 −25, ~+85 Elo (95%: +27…+148) |
| depth cap 32 against 12 | 2 s per move, ~130 games | ~60%, ~+78 Elo (roughly +26…+130) |

These numbers do not add up: the time controls differ, and self-play usually exaggerates a difference that an outside opponent would measure. What they do show is that at 2 seconds per move the current build is stronger than the one measured at ~2,585. It searches the same tree 1.42 times as fast, and without the 12-ply cap it uses the whole budget, where the earlier build used about half. How much stronger needs a new run against Stockfish.

According to Stockfish's own source, `UCI_Elo` is calibrated to roughly the CCRL Blitz engine list. That is not a human FIDE rating: there is no reliable conversion, and limited Stockfish does not make human mistakes. The honest reading of the table is *about 2,500 at 1 s per move and about 2,580 at 2 s, on limited Stockfish's scale, for the earlier build*.

The `2500` in the name `L2_aybars_2500` is a checkpoint label set before any absolute measurement; the measurement later landed close to it.

## How it was built

Aybars came out of a separate workbench with two rules: nothing enters unmeasured, and nothing measured goes unrecorded. Each change differed from the current base in exactly one place and was played against it in engine matches at 300 ms per move under SPRT (H0 0, H1 +20 Elo, α = β = 0.05, up to 1,500 games). Before any run, gates had to pass: perft on the rule layer; proof that the candidate differs from the base only where expected; the bot's move generator checked against the rule layer's; for a change meant only to be faster, identical node counts at fixed depth; and a short scan for errors. The byte cap was lifted along the way. The code is still written golfed, but a change is judged only by the Elo it brings and the time it costs.

The scope is a bot on top of an arbiter, not a new engine. The rule layer comes from FideLite verbatim and stays in sync with it: when FideLite's `L2` changes, its core is copied over again. The bot may depart from the rules' code as far as speed asks, and the rule layer stays off the search path.

## Files

| file | size | contents |
|---|---|---|
| `index.html` | 8,269 B | game, rules and bot; you play White. The same file as [`L2_aybars_2500.html`](https://github.com/cuneytinann/FideLite/blob/main/builds/special/L2_aybars_2500.html) on fidelite.art |
| `black.html` | 8,267 B | the same with you playing Black: the board turned (`w^7` instead of `w^112`) and the bot's colour `D` set from 0 (Black) to 8 (White) |
| `LICENSE` | | MIT License |

Zipped with zopfli (`advzip -z -4`), `index.html` comes to 4,972 bytes and `black.html` to 4,971; with `zip -9`, `index.html` is 5,108.

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
<summary><b>Bytes</b></summary>

The script is 7,997 bytes; the HTML around it is 272, the same as in FideLite's `L2.html`. By layer, as **Show code** on fidelite.art colours it: rules 1,241, interface 529, bot 6,227.

The rule layer against FideLite's `L2.html`, definition by definition, the trailing comma included:

| definition | `L2.html` | Aybars | note |
|---|---|---|---|
| opening array `b=[…]` | 33 | 35 | the interface's target square `f` |
| board setup `for(j in s=…)` | 81 | 81 | same |
| repetition table | 15 | 18 | `R=G[…]=1` against `R={[…]:$=1}` |
| `a=Math.abs` | — | 11 | for the bot |
| `C` · `G` · `T` · `U` · `L` | 29 · 237 · 53 · 131 · 55 | same | byte for byte |
| dead-position detector | `H` 20 + `N` 21 + `g` 134, the board scan in the driver | `l=(…)` 361, helpers local | |

The interface and driver come to 841 bytes in `L2.html` and 759 in Aybars (without the bot's hook in `A`, with the first drawing `d();`).

The bot, by part:

| part | bytes |
|---|---|
| search `Z` | 1,351 |
| evaluation `J0` (1,078), its side-to-move wrapper `j` (114), the cache `EC` (25) | 1,217 |
| move generator `g` with castling, direction tables `VD`, `VN` | 710 |
| static exchange `SE`, least valuable attacker `LV`, `SA`, `SO` | 588 |
| tables: piece values `P`, piece-square `K`, `PS`, phase `PH`, mobility weights `MW` | 563 |
| hash and undo: `F`, `S1`, `A1`, `A2`, `UN`, `SY`, `E1`, `M3`, `PT[0]` | 421 |
| driver `o` | 319 |
| make `M`, en-passant legality `EPL` | 314 |
| move ordering `EM`, move buffer `MV`, `S3`, `U1`, `U3`, `h` | 256 |
| check test `V`, king squares `KP` | 222 |
| other state: `D`, `O`, `X1`, `H`, `ID`, `B5`, `CM`, `TT`, `PT`, `Y2`, `W2`, `W3`, `FL` | 181 |
| hook that starts the bot after each move (`k=A,A=…`, the closing `setTimeout`) | 63 |
| inside `A`: `SY(),PT[++Y3]=H2,A1=0` (game history, undo log) | 22 |
| **total** | **6,227** |

Over FideLite's `L2.html` (1,922 bytes) Aybars adds 6,347: 6,227 of them are the bot, the remaining 120 the net of the small rule-side differences and a shorter, fixed-orientation interface. `black.html` is 2 bytes shorter (`w^7` against `w^112`).

</details>

<details>
<summary><b>Measured history</b></summary>

The largest measured steps on the way to the current build. Each row was measured against the base of its day, under the conditions shown; the rows do not add up.

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
| October 2026: the rule-layer shortenings carried over from FideLite (castling rook square, start-rank test, square colour, material weight) and three spellings in the bot's move generator (`16*w-8`, `6-5*w`) | −10 bytes; search results identical at fixed depths 4 and 6 over 8 positions, speed unchanged (−0.7%, within noise) |
| the 0x88 board, with a typed board copy, a ray-based check test and an evaluation cache in the bot | ×1.42 nodes per second; +85 (+27…+148), 100 games at 200 ms, both capped at 12 plies |
| depth cap 12 → 32 | ~+78 (roughly +26…+130), ~130 games at 2 s |

</details>

---

# Aybars (Türkçe)

[English](#aybars) · **Türkçe**

▶ **[Tarayıcıda hemen oyna](https://cuneytinann.github.io/Aybars_ChessAI/)**

**FideLite'ın kurallarının üstünde, 8.269 baytlık tek bir HTML dosyasında bir satranç botu.**

Aybars, [FideLite](https://www.fidelite.art/tr)'ın `L2` kural seviyesindeki hakeminin üstüne kurulmuş bir arama motoru. Sen beyazla, `black.html`'de siyahla oynuyorsun; Aybars cevap veriyor. Sayfanın tamamı (tahta, arayüz, kurallar ve bot) 8.269 bayt; zip'lenince 4.972. Önceki bir sürüm, gücü `UCI_Elo` ile sınırlanmış Stockfish'e karşı hamle başına 2 saniyede ~2.585'lik bir oyuncu gibi skor yaptı; bu ölçek insan FIDE puanı değil, CCRL Blitz motor listesine yakın. Bugünkü sürüm doğrudan maçlarda o sürümü geçiyor ([Güç](#güç)).

## Bir bakışta

| | |
|---|---|
| **Kurallar** | FideLite'ın `L2` kural çekirdeği, bayt bayt, 0x88 tahtada |
| **Bot** | bir arama (yinelemeli derinleştirmeli alfa-beta, transpozisyon tablosu, sessizlik araması) ve bir değerlendirme (materyal, taş-kare tabloları, piyon yapısı, hareketlilik, yalnız şahı mat etme) |
| **Boyut** | tek dosya, 8.269 bayt; zip'te 4.972 bayt. FideLite'ın `L2.html`'i 1.922 bayt; botun kendisi 6.227 |
| **Düşünme süresi** | hamle başına 2 saniyelik bütçe, ortalamada yaklaşık 1,8 sn kullanılıyor; en fazla 32 yarım hamle |
| **Oyun sonları** | mat, pat, ölü pozisyon (yetersiz materyal ya da kilitli piyonlar), üçlü tekrar, 50 hamle kuralı |
| **Siyahla oynamak** | `black.html`: tahta çevrilmiş ve ilk hamleyi Aybars yapıyor; iki değeri değiştirilmiş aynı dosya |
| **Kurulum** | yok: çevrimiçi oyna ya da bir dosyayı indirip aç |

## Hemen dene

1. Beyazla oynamak için [cuneytinann.github.io/Aybars_ChessAI](https://cuneytinann.github.io/Aybars_ChessAI/), siyahla oynamak için [black.html](https://cuneytinann.github.io/Aybars_ChessAI/black.html) adresini aç. İki dosyadan birini indirip yerelde de açabilirsin.
2. Kendi taşlarından birine, sonra yanan hedef karelerinden birine tıkla; Aybars bir iki saniye içinde cevap veriyor, `black.html`'de ilk hamleyi de o yapıyor. Terfi için tahtanın üstünde açılan düğmelerden taşı seç.
3. Durum satırında `M=` (50 hamle kuralına sayılan yarım hamleler), `R=` (mevcut pozisyonun kaçıncı kez geldiği) ve sırası gelen taraf şahtayken `C!` görünüyor. Oyun bitince yerine sonuç kodu geliyor: `W#` beyaz mat etti, `B#` siyah mat etti, `SM` pat, `DP` ölü pozisyon (yetersiz materyal ya da kilitli piyonlar), `3R` üçlü tekrar, `50` 50 hamle kuralı. Yeni oyun için sayfayı yenile.

2020 ve sonrasının herhangi bir tarayıcısı yeterli; sayfa BigInt istiyor (Chrome 67+, Firefox 68+, Safari 14+). Aynı dosya fidelite.art'ta [`L2_aybars_2500`](https://www.fidelite.art/tr#bots) adıyla sergileniyor; orada **Kodu göster** onu katmanlarına göre boyuyor: kurallar, arayüz ve bot.

## Kurallar: FideLite

Aybars kendi kuralını yazmıyor. Kural katmanı FideLite'ın `L2` seviyesi, [`builds/L2/L2.html`](https://github.com/cuneytinann/FideLite/blob/main/builds/L2/L2.html), olduğu gibi taşınmış: çekirdeği oluşturan beş fonksiyon bayt bayt aynı — `C` (bir karenin dokunduğu rok hakları), `G` (bir taş o hamleyi yapabilir mi), `T` (bir kare saldırı altında mı), `U` (hamleyi yapmak) ve `L` (yasallık). Yasal hamleler, rok, geçerken alma, terfi, şah, mat ve pat bunlardan geliyor; üstlerinde ölü pozisyon dedektörü (yetersiz materyal ve kilitli piyonlar, FIDE 5.2.2) ile tekrar ve 50 hamle sayaçları duruyor. Her kuralın nasıl okunup uygulandığı **[fidelite.art](https://www.fidelite.art/tr#rules)**'ta anlatılıyor; kural seviyelerinin kendisi [`L1` ve `L2`](https://www.fidelite.art/tr#levels) başlığında.

Kural katmanı tahtada oynanan hamleleri, hamle başına bir kez, yapıyor ve hükme bağlıyor; arama yolunda değil. Aybars'ın `L2`'den ayrıldığı yerler:

| | FideLite `L2` | Aybars |
|---|---|---|
| oyunun bittiği yer | beşli tekrar ve 75 hamle, kendiliğinden | üçlü tekrar ve 50 hamle (aşağıda) |
| tekrar tablosu | `R=G[…]=1`: `G` fonksiyonu aynı zamanda tablo, sayı `R`'de | `R={[…]:$=1}`, sayı `$`'da |
| ölü pozisyon dedektörü | `A`'nın içinde satır içi, tahta taraması sürücüde | aynı mantık `l()` fonksiyonuna sarılı, değişkenleri yerel; böylece adları botunkilerle çakışamıyor |
| eklenenler | — | bot için `a=Math.abs`; baş dizisinde arayüzün hedef karesi `f` |
| arayüz | tahta sırası gelen tarafa dönüyor; durum satırında koordinat var | senin rengin hep altta; koordinat yok; her hamlenden sonra bot oynuyor |

**Neden üçlü tekrar ve 50 hamle.** FideLite'ın `L2`'sinde oyuncu katmanı yok; talep eşikleri orada hiçbir şey tetiklemiyor ve oyun yalnızca kendiliğinden gelen beşli tekrar ve 75 hamlede bitiyor. Aybars bilerek üçlü tekrar ve 50 hamlede bitiyor: araması tekrarı ve 50 hamle sınırını beraberlik sayıyor, oyunun eşiğini yukarı çekip aramayı olduğu yerde bırakmak ikisini birbirinden koparırdı. FideLite bunu bilerek bırakılmış bir istisna olarak kayda geçiriyor.

## Bot

Script'te `U1=0`'dan sonuna kadar her şey, artı `A`'nın içinde her hamleyi bota ileten 22 baytlık kanca bot: toplam 6.227 bayt. Bot kural katmanıyla hiç kod paylaşmıyor. Hızın peşinden gitmekte serbest; bu yüzden tahtanın kendi kopyasını, kendi hamle üreticisini ve şah denetimini, kendi yap/geri al'ını taşıyor.

| parça | ne yapıyor |
|---|---|
| **Tahta** | bot her aramanın başında tahtanın tipli bir kopyasını alıyor (128 hücrelik `Int8Array`, tahta dışı hücreler = 7) ve sonda tahtayı geri koyuyor. Yazdığı her kare `S1`'den geçiyor; Zobrist hash'i, artımlı değerlendirme, oyun evresi, şah kareleri ve geri alma günlüğü böylece güncel kalıyor |
| **Hamle üretici** `g` | sözde-yasal; rok (hak, boş kareler, şahın durduğu ve geçtiği karenin saldırı altında olmaması) ve geçerken almanın yasallığı (`EPL`) içinde. Kuralsız bir hamle çocuk düğümün başında yakalanıyor: sırası gelen taraf şahı alabiliyorsa düğüm hemen dönüyor. Vezire ya da ata terfi ediyor |
| **Şah denetimi** `V` | şahın karesinden dışarı ışınlar: sekiz yön ve sekiz at sıçraması; piyon ve şah ışının ilk karesinde sınanıyor |
| **Arama** `Z` | negamax alfa-beta; transpozisyon tablosu (32 bitlik Zobrist hash'i, tek bir `Int32Array`'de üçer sayılık 2²¹ girdi, hamleden hamleye korunuyor); alışlar ve terfi itişleri üzerinde sessizlik araması (stand-pat, delta budaması ve kaybettiren alışların SEE ile budanması); 1–3 derinlikte ters futility budaması (yarım hamle başına 120 cp, şah çekilmiyorken); sessiz hamlelerde geç hamle indirimi (LMR): üçüncüden itibaren bir, 4'ün üstündeki derinlikte altıncıdan itibaren iki, 6'nın üstündeki derinlikte on birinciden itibaren üç yarım hamle |
| **Hamle sıralama** `EM` | önce hash hamlesi, sonra kurban ve saldırana göre alışlar (MVV-LVA), terfiler, killer hamle, karşı hamle (counter-move) ve geçmiş tablosu |
| **Sürücü** `o` | 2 saniye içinde 32 yarım hamleye kadar yinelemeli derinleştirme; üçüncü turdan itibaren ±40 cp'lik aspirasyon penceresi; yeni tur ancak geçen süre artı son turun iki katı bütçeye sığarsa başlıyor |
| **Değerlendirme** `j` | materyal ve taş-kare tabloları, artımlı olarak tutuluyor, 2¹⁸ sonuçluk bir önbellekle; geçer piyon (sıra² × 4, önü kapalıysa yarısı) ve izole piyon (−25); açık ve yarı açık hattaki kaleler; hareketlilik; taşların rakip şaha yakınlığı; şahın tablosu materyale göre orta oyundan oyunsonuna harmanlanıyor; bir tarafta şahtan başka taş kalmayınca o şahı kenara (fil ve atla, filin rengindeki köşeye) sürmek ve şahları yaklaştırmak |
| **Beraberlikler** | oyun geçmişine ve arama yoluna karşı tekrarlar, 50 hamle sınırı; beraberlik bot için −20 cp sayılıyor, böylece imkânı varken oynamaya devam ediyor |

Aramanın içinde süre denetimi yok; tahminden uzun süren bir tur hamleyi bütçenin dışına taşırıyor: yavaş bir makinede 28 deneme pozisyonunun 4'ü 2 saniyeyi aştı, en uzunu 7 saniye sürdü.

Taş değerleri AlphaZero'nun Tomašev ve arkadaşlarının *Assessing Game Balance with AlphaZero* çalışmasındaki kestirimlerinden geliyor (piyon 100, at 305, fil 333, vezir 950); kale 563'ten 489'a indirildi. Taş-kare tabloları Tomasz Michniewski'nin Simplified Evaluation Function tablolarından başlıyor ve Zurichess'in quiet-labeled pozisyonları üzerinde Texel ayarıyla yeniden uyduruldu.

## Güç

Aybars, `UCI_Elo` ile gücü sınırlanmış Stockfish 17.1'e karşı ölçüldü; Stockfish hamle başına 300 ms düşündü. Sınırlı Stockfish hamlesini sabit ve sığ bir derinlikte seçtiği için gücü süreye neredeyse hiç bağlı değil; Aybars'ınki bağlı. Ölçüm önceki bir sürümde yapıldı: aynı arama ve değerlendirme, 64 karelik bir tahtada, derinliği 12 yarım hamleyle sınırlı. Her satır 100 maç, ±35.

| önceki sürümün hamle başına süresi | Stockfish `UCI_Elo` | skor | tahmin |
|---|---|---|---|
| 0,3 sn | 2400 / 2500 | %55,5 / %44 | ~2.445 |
| 1 sn | 2500 | %53 | ~2.520 |
| 2 sn | 2500 | %62 | ~2.585 |

Bugünkü sürüm Stockfish'e karşı yeniden ölçülmedi. Önceki sürüme karşı:

| maç | koşul | sonuç |
|---|---|---|
| 0x88 tahta, 64 karelik tahtaya karşı; ikisi de 12 yarım hamleyle sınırlı | hamle başına 200 ms, 100 maç | +49 =26 −25, ~+85 Elo (%95: +27…+148) |
| derinlik tavanı 32, 12'ye karşı | hamle başına 2 sn, ~130 maç | ~%60, ~+78 Elo (kabaca +26…+130) |

Bu sayılar toplanmaz: süre koşulları farklı ve kendi kendine maç, dış bir rakibin ölçeceği farkı genelde büyütür. Gösterdikleri şu: hamle başına 2 saniyede bugünkü sürüm, ~2.585 ölçülen sürümden güçlü. Aynı ağacı 1,42 kat hızlı geziyor ve 12 yarım hamle tavanı kalktığı için bütçenin tamamını kullanıyor; önceki sürüm yaklaşık yarısını kullanıyordu. Ne kadar güçlü olduğu Stockfish'e karşı yeni bir ölçüm istiyor.

Stockfish'in kendi kaynağına göre `UCI_Elo` kabaca CCRL Blitz motor listesine göre ayarlanmış. Bu bir insan FIDE puanı değil: güvenilir bir çevrim yok, sınırlı Stockfish de insan gibi hata yapmıyor. Tablonun dürüst okuması şu: *önceki sürüm için, sınırlı Stockfish'in ölçeğinde hamle başına 1 sn'de yaklaşık 2.500, 2 sn'de yaklaşık 2.580*.

`L2_aybars_2500` adındaki `2500`, hiçbir mutlak ölçüm yapılmadan konmuş bir durak etiketi; ölçüm sonradan ona yakın çıktı.

## Nasıl yapıldı

Aybars, iki kuralı olan ayrı bir tezgâhtan çıktı: ölçülmemiş hiçbir şey girmez, ölçülmüş hiçbir şey kaydedilmeden bırakılmaz. Her değişiklik mevcut tabandan tek bir yerde ayrıldı ve ona karşı, hamle başına 300 ms'lik motor maçlarında SPRT ile oynatıldı (H0 0, H1 +20 Elo, α = β = 0,05, en fazla 1.500 maç). Her koşudan önce kapılar geçmek zorundaydı: kural katmanında perft; adayın tabandan yalnızca beklenen yerde ayrıldığının kanıtı; botun hamle üreticisinin kural katmanınınkine karşı sınanması; yalnız hız için yapılan bir değişiklikte sabit derinlikte birebir aynı düğüm sayısı; ve kısa bir hata taraması. Bayt tavanı yol üstünde kaldırıldı. Kod hâlâ golf üslubuyla yazılıyor, ama bir değişiklik yalnızca getirdiği Elo ve harcattığı zamanla değerlendiriliyor.

Kapsam bir hakemin üstüne bot, yeni bir motor değil. Kural katmanı FideLite'tan olduğu gibi geliyor ve onunla senkron kalıyor: FideLite'ın `L2`'si değişince çekirdeği yeniden kopyalanıyor. Bot, hız ne kadar istiyorsa kuralların kodundan o kadar ayrılabiliyor; kural katmanı da arama yolunun dışında kalıyor.

## Dosyalar

| dosya | boyut | içerik |
|---|---|---|
| `index.html` | 8.269 B | oyun, kurallar ve bot; sen beyazla oynuyorsun. fidelite.art'taki [`L2_aybars_2500.html`](https://github.com/cuneytinann/FideLite/blob/main/builds/special/L2_aybars_2500.html) ile aynı dosya |
| `black.html` | 8.267 B | sen siyahla oynarken aynısı: tahta çevrilmiş (`w^112` yerine `w^7`) ve botun rengi `D` 0'dan (siyah) 8'e (beyaz) alınmış |
| `LICENSE` | | MIT Lisansı |

Zopfli ile (`advzip -z -4`) zip'lenince `index.html` 4.972, `black.html` 4.971 bayt tutuyor; `zip -9` ile `index.html` 5.108.

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
<summary><b>Baytlar</b></summary>

Script 7.997 bayt; çevresindeki HTML 272 bayt, FideLite'ın `L2.html`'indekiyle aynı. Katmanlara göre, fidelite.art'taki **Kodu göster**'in boyadığı gibi: kurallar 1.241, arayüz 529, bot 6.227.

Kural katmanı, FideLite'ın `L2.html`'ine karşı tanım tanım, sondaki virgül dahil:

| tanım | `L2.html` | Aybars | not |
|---|---|---|---|
| baş dizisi `b=[…]` | 33 | 35 | arayüzün hedef karesi `f` |
| tahta kurulumu `for(j in s=…)` | 81 | 81 | aynı |
| tekrar tablosu | 15 | 18 | `R=G[…]=1`'e karşı `R={[…]:$=1}` |
| `a=Math.abs` | — | 11 | bot için |
| `C` · `G` · `T` · `U` · `L` | 29 · 237 · 53 · 131 · 55 | aynı | bayt bayt |
| ölü pozisyon dedektörü | `H` 20 + `N` 21 + `g` 134, tahta taraması sürücüde | `l=(…)` 361, yardımcılar yerel | |

Arayüz ve sürücü `L2.html`'de 841, Aybars'ta 759 bayt (`A`'daki bot kancası hariç, ilk çizim `d();` dahil).

Bot, parça parça:

| parça | bayt |
|---|---|
| arama `Z` | 1.351 |
| değerlendirme `J0` (1.078), sırası gelen tarafa çeviren `j` (114), önbellek `EC` (25) | 1.217 |
| rok dahil hamle üretici `g`, yön tabloları `VD`, `VN` | 710 |
| statik alış değerlendirmesi `SE`, en ucuz saldıran `LV`, `SA`, `SO` | 588 |
| tablolar: taş değerleri `P`, taş-kare `K`, `PS`, evre `PH`, hareketlilik ağırlıkları `MW` | 563 |
| hash ve geri alma: `F`, `S1`, `A1`, `A2`, `UN`, `SY`, `E1`, `M3`, `PT[0]` | 421 |
| sürücü `o` | 319 |
| yap `M`, geçerken alma yasallığı `EPL` | 314 |
| hamle sıralama `EM`, hamle tamponu `MV`, `S3`, `U1`, `U3`, `h` | 256 |
| şah denetimi `V`, şah kareleri `KP` | 222 |
| öteki durum: `D`, `O`, `X1`, `H`, `ID`, `B5`, `CM`, `TT`, `PT`, `Y2`, `W2`, `W3`, `FL` | 181 |
| her hamleden sonra botu başlatan kanca (`k=A,A=…`, sondaki `setTimeout`) | 63 |
| `A`'nın içinde: `SY(),PT[++Y3]=H2,A1=0` (oyun geçmişi, geri alma günlüğü) | 22 |
| **toplam** | **6.227** |

Aybars, FideLite'ın `L2.html`'ine (1.922 bayt) 6.347 bayt ekliyor: 6.227'si bot, kalan 120'si kural tarafındaki küçük farklarla daha kısa, sabit yönlü arayüzün net toplamı. `black.html` 2 bayt kısa (`w^112` yerine `w^7`).

</details>

<details>
<summary><b>Ölçüm geçmişi</b></summary>

Bugünkü sürüme giden yoldaki en büyük ölçülmüş adımlar. Her satır kendi gününün tabanına karşı, gösterilen koşullarda ölçüldü; satırlar toplanmaz.

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
| Ekim 2026: FideLite'tan taşınan kural katmanı kısaltmaları (rok kalesinin karesi, başlangıç yatayı testi, kare rengi, materyal ağırlığı) ve botun hamle üreticisinde üç yazım (`16*w-8`, `6-5*w`) | −10 bayt; 8 pozisyonda sabit 4 ve 6 derinlikte arama sonuçları birebir aynı, hız değişmedi (−%0,7, gürültü sınırında) |
| 0x88 tahta; botta tipli tahta kopyası, ışınla şah denetimi ve değerlendirme önbelleği | saniyede ×1,42 düğüm; +85 (+27…+148), 200 ms'de 100 maç, ikisi de 12 yarım hamleyle sınırlı |
| derinlik tavanı 12 → 32 | ~+78 (kabaca +26…+130), 2 sn'de ~130 maç |

</details>
