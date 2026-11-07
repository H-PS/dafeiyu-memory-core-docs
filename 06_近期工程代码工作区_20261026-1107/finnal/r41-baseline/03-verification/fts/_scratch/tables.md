# tables from run-20261105T121058.json  (run_id 20261105T121058, 77489ddec253…)

## T1  per-query expected-vs-actual

Cell = `returned/|Rel|` with `✓` when every relevant doc was retrieved, `·` when at least one
relevant doc was retrieved but not all, `✗` when none was. `-` = |Rel| = 0 (no recall value).

| query | kind | |Rel| | u61 | ascii | tri | tri+LIKE | bigram | uni+bi | LIKE |
|---|---|---|---|---|---|---|---|---|---|
| q01 `中文` | chinese_substring | 6 | `2/6` · | `2/6` · | `0/6` ✗ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ |
| q02 `记忆库` | chinese_substring | 2 | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ |
| q03 `中文测试内容` | chinese_substring | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q04 `爵士乐` | chinese_substring | 1 | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q05 `旅行` | chinese_substring | 2 | `2/2` ✓ | `2/2` ✓ | `0/2` ✗ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ |
| q06 `项目` | chinese_substring | 6 | `6/6` ✓ | `6/6` ✓ | `0/6` ✗ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ |
| q07 `咖啡` | chinese_substring | 3 | `0/3` ✗ | `0/3` ✗ | `0/3` ✗ | `3/3` ✓ | `3/3` ✓ | `3/3` ✓ | `3/3` ✓ |
| q08 `苹果` | chinese_substring | 2 | `1/2` · | `1/2` · | `0/2` ✗ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ | `2/2` ✓ |
| q09 `鸡蛋` | chinese_substring | 1 | `1/1` ✓ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q10 `测试` | chinese_substring | 10 | `4/10` · | `4/10` · | `0/10` ✗ | `10/10` ✓ | `10/10` ✓ | `10/10` ✓ | `10/10` ✓ |
| q11 `检索` | chinese_substring | 6 | `3/6` · | `3/6` · | `0/6` ✗ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ |
| q12 `西` | chinese_single_char | 1 | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ |
| q13 `我` | chinese_single_char | 6 | `2/6` · | `2/6` · | `0/6` ✗ | `6/6` ✓ | `2/6` · | `6/6` ✓ | `6/6` ✓ |
| q14 `的` | chinese_single_char | 11 | `3/11` · | `3/11` · | `0/11` ✗ | `11/11` ✓ | `3/11` · | `11/11` ✓ | `11/11` ✓ |
| q15 `爱` | chinese_single_char | 1 | `1/1` ✓ | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q16 `库` | chinese_single_char | 5 | `2/5` · | `1/5` · | `0/5` ✗ | `5/5` ✓ | `2/5` · | `5/5` ✓ | `5/5` ✓ |
| q17 `mixed` | mixed | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q18 `FTS5` | abbreviation | 4 | `4/4` ✓ | `3/4` · | `4/4` ✓ | `4/4` ✓ | `4/4` ✓ | `4/4` ✓ | `4/4` ✓ |
| q19 `bge-m3` | abbreviation | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q20 `user_id` | special_symbol | 1 | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q21 `snake_case` | special_symbol | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q22 `MacBook` | english | 1 | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q23 `macbook` | english | 1 | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q24 `weekly` | english | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q25 `reimbursement` | english | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q26 `8080` | digits | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q27 `2026` | digits | 2 | `1/2` · | `1/2` · | `2/2` ✓ | `2/2` ✓ | `1/2` · | `1/2` · | `2/2` ✓ |
| q28 `1000` | digits | 1 | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q29 `繁體` | unicode_edge | 1 | `1/1` ✓ | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q30 `繁体` | unicode_edge | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ |
| q31 `Café` | unicode_edge | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q32 `Cafe` | unicode_edge | 1 | `1/1` ✓ | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `0/1` ✗ |
| q33 `🍰` | unicode_edge | 1 | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ |
| q34 `１２３` | unicode_edge | 1 | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q35 `123` | unicode_edge | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ |
| q36 `𠀀` | unicode_edge | 1 | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ |
| q37 `、` | chinese_punctuation | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `4` **FP** | `0` ✓ | `0` ✓ | `4` **FP** |
| q38 `——` | chinese_punctuation | 0 | `0` ✓ | `1` **FP** | `0` ✓ | `3` **FP** | `0` ✓ | `0` ✓ | `3` **FP** |
| q39 `紫罗兰` | negative_control | 1 | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q40 `加密` | negative_control | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ |
| q41 `浏览` | negative_control | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ |
| q42 `不存在的词汇组合xyz` | negative_control | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ |
| q43 `项目*` | prefix | 6 | `6/6` ✓ | `6/6` ✓ | `0/6` ✗ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ | `6/6` ✓ |
| q44 `检索*` | prefix | 3 | `3/3` ✓ | `3/3` ✓ | `0/3` ✗ | `1/3` · | `6/3` ·+3FP | `6/3` ·+3FP | `1/3` · |
| q45 `"西湖 散步"` | phrase | 1 | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q46 `"记忆 库"` | phrase | 1 | `2/1` ·+1FP | `1/1` ✓ | `1/1` ✓ | `2/1` ✗+2FP | `2/1` ✗+2FP | `2/1` ✗+2FP | `5/1` ·+4FP |
| q47 `咖啡 AND 拿铁` | boolean | 1 | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `3/1` ·+2FP |
| q48 `苹果 OR 咖啡` | boolean | 5 | `1/5` · | `1/5` · | `0/5` ✗ | `0/5` ✗ | `5/5` ✓ | `5/5` ✓ | `2/5` · |
| q49 `` | empty_query | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ |
| q50 `   ` | empty_query | 0 | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ | `0` ✓ |
| q51 `"中文" AND (测试 OR 检索) NOT 排除` | injection | 0 | `1` **FP** | `1` **FP** | `0` ✓ | `0` ✓ | `3` **FP** | `3` **FP** | `6` **FP** |
| q52 `我记得你上次说过你喜欢在雨天听爵士乐并且不喜欢被人打扰所以我一直没有` | long_query | 1 | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q53 `记忆` | chinese_substring | 5 | `3/5` · | `2/5` · | `0/5` ✗ | `5/5` ✓ | `5/5` ✓ | `5/5` ✓ | `5/5` ✓ |
| q54 `分词` | chinese_substring | 1 | `1/1` ✓ | `1/1` ✓ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |
| q55 `蛋糕` | chinese_substring | 1 | `0/1` ✗ | `0/1` ✗ | `0/1` ✗ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ | `1/1` ✓ |

## T2  aggregate per config

| config | hit@10 | full-recall | recall@5 | recall@10 | prec@5 | prec@10 | FP | FN | zero-result | errors | p50 ms | p95 ms | fts index B |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| unicode61-default | 0.8 | 0.5778 | 0.6601 | 0.6675 | 0.9861 | 0.9861 | 1 | 47 | 9 | 0 | 0.0181 | 0.0394 | 36864 |
| unicode61-rd0 | 0.7778 | 0.5556 | 0.6379 | 0.6453 | 0.9857 | 0.9857 | 1 | 48 | 10 | 0 | 0.0178 | 0.0416 | 36864 |
| unicode61-rd2 | 0.8 | 0.5778 | 0.6601 | 0.6675 | 0.9861 | 0.9861 | 1 | 47 | 9 | 0 | 0.0161 | 0.0261 | 36864 |
| unicode61-tokenchars-underscore | 0.8 | 0.5778 | 0.6601 | 0.6675 | 0.9861 | 0.9861 | 1 | 47 | 9 | 0 | 0.0169 | 0.035 | 40960 |
| unicode61-separators-middot | 0.8 | 0.5778 | 0.6601 | 0.6675 | 0.9861 | 0.9861 | 1 | 47 | 9 | 0 | 0.0157 | 0.0276 | 36864 |
| ascii | 0.6444 | 0.4 | 0.4901 | 0.4975 | 1 | 1 | 0 | 57 | 16 | 0 | 0.015 | 0.0287 | 40960 |
| porter | 0.8 | 0.5778 | 0.6601 | 0.6675 | 0.9861 | 0.9861 | 1 | 47 | 9 | 0 | 0.0165 | 0.0394 | 40960 |
| trigram | 0.4444 | 0.4444 | 0.4444 | 0.4444 | 1 | 1 | 0 | 87 | 25 | 0 | 0.0136 | 0.0362 | 57344 |
| trigram+like-fallback | 0.9111 | 0.8889 | 0.8545 | 0.8943 | 0.9762 | 0.9762 | 2 | 10 | 3 | 0 | 0.0239 | 0.0424 | 57344 |
| unicode61+bigram-shadow | 0.9111 | 0.8222 | 0.8224 | 0.8557 | 0.9619 | 0.9643 | 5 | 20 | 3 | 0 | 0.0187 | 0.0406 | 49152 |
| unicode61+unigram-bigram-shadow | 0.9556 | 0.9333 | 0.8953 | 0.9424 | 0.9636 | 0.9659 | 5 | 3 | 1 | 0 | 0.0167 | 0.0387 | 61440 |
| unicode61-like-substring | 0.9778 | 0.9333 | 0.9079 | 0.9476 | 0.9667 | 0.9667 | 6 | 6 | 1 | 0 | 0.0237 | 0.0398 | 36864 |

## T3  per-kind hit-rate@10 and recall@10

| kind | n | u61 hit/rec | ascii hit/rec | tri hit/rec | tri+LIKE hit/rec | bigram hit/rec | uni+bi hit/rec | LIKE hit/rec |
|---|---|---|---|---|---|---|---|---|
| chinese_substring | 14 | 0.7857 / 0.5952 | 0.7143 / 0.5095 | 0.2143 / 0.2143 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 |
| chinese_single_char | 5 | 0.8 / 0.4012 | 0.8 / 0.3612 | 0 / 0 | 1 / 0.9818 | 0.8 / 0.4012 | 1 / 0.9818 | 1 / 0.9818 |
| mixed | 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 |
| abbreviation | 2 | 1 / 1 | 1 / 0.875 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 |
| special_symbol | 2 | 1 / 1 | 0.5 / 0.5 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 |
| english | 4 | 1 / 1 | 0.5 / 0.5 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 |
| digits | 3 | 1 / 0.8333 | 0.6667 / 0.5 | 1 / 1 | 1 / 1 | 1 / 0.8333 | 1 / 0.8333 | 1 / 1 |
| unicode_edge | 6 | 0.6667 / 0.6667 | 0.3333 / 0.3333 | 0.3333 / 0.3333 | 0.8333 / 0.8333 | 0.6667 / 0.6667 | 0.8333 / 0.8333 | 0.8333 / 0.8333 |
| chinese_punctuation | 0 | — | — | — | — | — | — | — |
| negative_control | 1 | 0 / 0 | 0 / 0 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 |
| prefix | 2 | 1 / 1 | 1 / 1 | 0 / 0 | 1 / 0.6666 | 1 / 1 | 1 / 1 | 1 / 0.6666 |
| phrase | 2 | 0.5 / 0.5 | 0.5 / 0.5 | 0.5 / 0.5 | 0.5 / 0.5 | 0.5 / 0.5 | 0.5 / 0.5 | 1 / 1 |
| boolean | 2 | 0.5 / 0.1 | 0.5 / 0.1 | 0 / 0 | 0 / 0 | 1 / 1 | 1 / 1 | 1 / 0.7 |
| empty_query | 0 | — | — | — | — | — | — | — |
| injection | 0 | — | — | — | — | — | — | — |
| long_query | 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 | 1 / 1 |

## T4  scale corpus (synthetic, unlabelled, n=20000)

| config | build ms | fts index B | db file B | B/doc | p50 ms | p95 ms | max ms | worst-query p50 ms |
|---|---|---|---|---|---|---|---|---|
| unicode61-default | 230.9 | 8990720 | 16396288 | 449.5 | 0.0393 | 2.2751 | 2.5913 | 2.3127 |
| unicode61-rd0 | 225.15 | 8990720 | 16396288 | 449.5 | 0.0315 | 2.4593 | 2.8445 | 2.5258 |
| unicode61-rd2 | 233.66 | 8990720 | 16396288 | 449.5 | 0.0714 | 2.0987 | 2.3962 | 2.1145 |
| unicode61-tokenchars-underscore | 229.98 | 8974336 | 16379904 | 448.7 | 0.0293 | 2.3272 | 2.8733 | 2.3336 |
| unicode61-separators-middot | 235.11 | 8990720 | 16396288 | 449.5 | 0.0316 | 2.2961 | 3.5913 | 2.3528 |
| ascii | 236.41 | 9039872 | 16556032 | 452 | 0.0248 | 1.4435 | 1.5873 | 1.4577 |
| porter | 246.08 | 8990720 | 16396288 | 449.5 | 0.0229 | 2.1787 | 2.4074 | 2.1816 |
| trigram | 463.1 | 18923520 | 27111424 | 946.2 | 0.0193 | 1.4753 | 1.7886 | 1.4905 |
| trigram+like-fallback | 475.32 | 18923520 | 27111424 | 946.2 | 0.8624 | 13.9424 | 15.634 | 14.0014 |
| unicode61+bigram-shadow | 497.02 | 15941632 | 23347200 | 797.1 | 0.9831 | 3.68 | 4.0355 | 3.6157 |
| unicode61+unigram-bigram-shadow | 730.49 | 27193344 | 36036608 | 1359.7 | 1.0216 | 3.9128 | 4.2786 | 3.9376 |
| unicode61-like-substring | 227.39 | 8990720 | 16396288 | 449.5 | 0.2651 | 13.6886 | 15.0258 | 13.7221 |

## T5  false-positive / false-negative detail

### unicode61-default

- q01 `中文` [chinese_substring] rel=6 returned=2  FN=[d04,d09,d11,d26]
- q04 `爵士乐` [chinese_substring] rel=1 returned=0  FN=[d20]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=1  FN=[d15]
- q10 `测试` [chinese_substring] rel=10 returned=4  FN=[d04,d12,d18,d24,d29,d43]
- q11 `检索` [chinese_substring] rel=6 returned=3  FN=[d04,d14,d17]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=2  FN=[d32,d40,d41]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q39 `紫罗兰` [negative_control] rel=1 returned=0  FN=[d17]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FP=[d12]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=1  FN=[d02,d08,d15,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=1  FP=[d42]
- q53 `记忆` [chinese_substring] rel=5 returned=3  FN=[d40,d41]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### unicode61-rd0

- q01 `中文` [chinese_substring] rel=6 returned=2  FN=[d04,d09,d11,d26]
- q04 `爵士乐` [chinese_substring] rel=1 returned=0  FN=[d20]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=1  FN=[d15]
- q10 `测试` [chinese_substring] rel=10 returned=4  FN=[d04,d12,d18,d24,d29,d43]
- q11 `检索` [chinese_substring] rel=6 returned=3  FN=[d04,d14,d17]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=2  FN=[d32,d40,d41]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q32 `Cafe` [unicode_edge] rel=1 returned=0  FN=[d28]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q39 `紫罗兰` [negative_control] rel=1 returned=0  FN=[d17]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FP=[d12]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=1  FN=[d02,d08,d15,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=1  FP=[d42]
- q53 `记忆` [chinese_substring] rel=5 returned=3  FN=[d40,d41]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### unicode61-rd2

- q01 `中文` [chinese_substring] rel=6 returned=2  FN=[d04,d09,d11,d26]
- q04 `爵士乐` [chinese_substring] rel=1 returned=0  FN=[d20]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=1  FN=[d15]
- q10 `测试` [chinese_substring] rel=10 returned=4  FN=[d04,d12,d18,d24,d29,d43]
- q11 `检索` [chinese_substring] rel=6 returned=3  FN=[d04,d14,d17]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=2  FN=[d32,d40,d41]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q39 `紫罗兰` [negative_control] rel=1 returned=0  FN=[d17]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FP=[d12]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=1  FN=[d02,d08,d15,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=1  FP=[d42]
- q53 `记忆` [chinese_substring] rel=5 returned=3  FN=[d40,d41]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### unicode61-tokenchars-underscore

- q01 `中文` [chinese_substring] rel=6 returned=2  FN=[d04,d09,d11,d26]
- q04 `爵士乐` [chinese_substring] rel=1 returned=0  FN=[d20]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=1  FN=[d15]
- q10 `测试` [chinese_substring] rel=10 returned=4  FN=[d04,d12,d18,d24,d29,d43]
- q11 `检索` [chinese_substring] rel=6 returned=3  FN=[d04,d14,d17]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=2  FN=[d32,d40,d41]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q39 `紫罗兰` [negative_control] rel=1 returned=0  FN=[d17]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FP=[d12]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=1  FN=[d02,d08,d15,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=1  FP=[d42]
- q53 `记忆` [chinese_substring] rel=5 returned=3  FN=[d40,d41]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### unicode61-separators-middot

- q01 `中文` [chinese_substring] rel=6 returned=2  FN=[d04,d09,d11,d26]
- q04 `爵士乐` [chinese_substring] rel=1 returned=0  FN=[d20]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=1  FN=[d15]
- q10 `测试` [chinese_substring] rel=10 returned=4  FN=[d04,d12,d18,d24,d29,d43]
- q11 `检索` [chinese_substring] rel=6 returned=3  FN=[d04,d14,d17]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=2  FN=[d32,d40,d41]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q39 `紫罗兰` [negative_control] rel=1 returned=0  FN=[d17]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FP=[d12]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=1  FN=[d02,d08,d15,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=1  FP=[d42]
- q53 `记忆` [chinese_substring] rel=5 returned=3  FN=[d40,d41]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### ascii

- q01 `中文` [chinese_substring] rel=6 returned=2  FN=[d04,d09,d11,d26]
- q04 `爵士乐` [chinese_substring] rel=1 returned=0  FN=[d20]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=1  FN=[d15]
- q09 `鸡蛋` [chinese_substring] rel=1 returned=0  FN=[d08]
- q10 `测试` [chinese_substring] rel=10 returned=4  FN=[d04,d12,d18,d24,d29,d43]
- q11 `检索` [chinese_substring] rel=6 returned=3  FN=[d04,d14,d17]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=1  FN=[d12,d32,d40,d41]
- q18 `FTS5` [abbreviation] rel=4 returned=3  FN=[d37]
- q20 `user_id` [special_symbol] rel=1 returned=0  FN=[d27]
- q22 `MacBook` [english] rel=1 returned=0  FN=[d16]
- q23 `macbook` [english] rel=1 returned=0  FN=[d16]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q28 `1000` [digits] rel=1 returned=0  FN=[d39]
- q32 `Cafe` [unicode_edge] rel=1 returned=0  FN=[d28]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q34 `１２３` [unicode_edge] rel=1 returned=0  FN=[d43]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q38 `——` [chinese_punctuation] rel=0 returned=1  FP=[d13]
- q39 `紫罗兰` [negative_control] rel=1 returned=0  FN=[d17]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=1  FN=[d02,d08,d15,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=1  FP=[d42]
- q53 `记忆` [chinese_substring] rel=5 returned=2  FN=[d12,d40,d41]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### porter

- q01 `中文` [chinese_substring] rel=6 returned=2  FN=[d04,d09,d11,d26]
- q04 `爵士乐` [chinese_substring] rel=1 returned=0  FN=[d20]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=1  FN=[d15]
- q10 `测试` [chinese_substring] rel=10 returned=4  FN=[d04,d12,d18,d24,d29,d43]
- q11 `检索` [chinese_substring] rel=6 returned=3  FN=[d04,d14,d17]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=2  FN=[d32,d40,d41]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q39 `紫罗兰` [negative_control] rel=1 returned=0  FN=[d17]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FP=[d12]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=1  FN=[d02,d08,d15,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=1  FP=[d42]
- q53 `记忆` [chinese_substring] rel=5 returned=3  FN=[d40,d41]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### trigram

- q01 `中文` [chinese_substring] rel=6 returned=0  FN=[d04,d09,d11,d13,d26,d42]
- q05 `旅行` [chinese_substring] rel=2 returned=0  FN=[d21,d22]
- q06 `项目` [chinese_substring] rel=6 returned=0  FN=[d03,d31,d32,d33,d34,d35]
- q07 `咖啡` [chinese_substring] rel=3 returned=0  FN=[d02,d08,d36]
- q08 `苹果` [chinese_substring] rel=2 returned=0  FN=[d15,d16]
- q09 `鸡蛋` [chinese_substring] rel=1 returned=0  FN=[d08]
- q10 `测试` [chinese_substring] rel=10 returned=0  FN=[d04,d09,d12,d13,d18,d23,d24,d29,d42,d43]
- q11 `检索` [chinese_substring] rel=6 returned=0  FN=[d04,d13,d14,d17,d37,d42]
- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=0  FN=[d01,d02,d04,d20,d23,d45]
- q14 `的` [chinese_single_char] rel=11 returned=0  FN=[d01,d02,d03,d04,d07,d14,d20,d22,d28,d40,d42]
- q15 `爱` [chinese_single_char] rel=1 returned=0  FN=[d23]
- q16 `库` [chinese_single_char] rel=5 returned=0  FN=[d12,d32,d40,d41,d42]
- q29 `繁體` [unicode_edge] rel=1 returned=0  FN=[d11]
- q32 `Cafe` [unicode_edge] rel=1 returned=0  FN=[d28]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q43 `项目*` [prefix] rel=6 returned=0  FN=[d03,d31,d32,d33,d34,d35]
- q44 `检索*` [prefix] rel=3 returned=0  FN=[d13,d37,d42]
- q45 `"西湖 散步"` [phrase] rel=1 returned=0  FN=[d01]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=0  FN=[d02,d08,d15,d16,d36]
- q53 `记忆` [chinese_substring] rel=5 returned=0  FN=[d12,d20,d40,d41,d42]
- q54 `分词` [chinese_substring] rel=1 returned=0  FN=[d42]
- q55 `蛋糕` [chinese_substring] rel=1 returned=0  FN=[d10]

### trigram+like-fallback

- q32 `Cafe` [unicode_edge] rel=1 returned=0  FN=[d28]
- q37 `、` [chinese_punctuation] rel=0 returned=4  FP=[d17,d26,d08,d34]
- q38 `——` [chinese_punctuation] rel=0 returned=3  FP=[d26,d08,d13]
- q44 `检索*` [prefix] rel=3 returned=1  FN=[d13,d42]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FN=[d42]  FP=[d41,d40]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=0  FN=[d02]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=0  FN=[d02,d08,d15,d16,d36]

### unicode61+bigram-shadow

- q12 `西` [chinese_single_char] rel=1 returned=0  FN=[d01]
- q13 `我` [chinese_single_char] rel=6 returned=2  FN=[d01,d02,d04,d20]
- q14 `的` [chinese_single_char] rel=11 returned=3  FN=[d01,d02,d03,d07,d20,d22,d40,d42]
- q16 `库` [chinese_single_char] rel=5 returned=2  FN=[d32,d40,d41]
- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q36 `𠀀` [unicode_edge] rel=1 returned=0  FN=[d29]
- q44 `检索*` [prefix] rel=3 returned=6  FP=[d14,d04,d17]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FN=[d42]  FP=[d40,d41]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=3  FP=[d04,d09,d42]

### unicode61+unigram-bigram-shadow

- q27 `2026` [digits] rel=2 returned=1  FN=[d06]
- q33 `🍰` [unicode_edge] rel=1 returned=0  FN=[d10]
- q44 `检索*` [prefix] rel=3 returned=6  FP=[d14,d04,d17]
- q46 `"记忆 库"` [phrase] rel=1 returned=2  FN=[d42]  FP=[d40,d41]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=3  FP=[d04,d09,d42]

### unicode61-like-substring

- q32 `Cafe` [unicode_edge] rel=1 returned=0  FN=[d28]
- q37 `、` [chinese_punctuation] rel=0 returned=4  FP=[d17,d26,d08,d34]
- q38 `——` [chinese_punctuation] rel=0 returned=3  FP=[d26,d08,d13]
- q44 `检索*` [prefix] rel=3 returned=1  FN=[d13,d42]
- q46 `"记忆 库"` [phrase] rel=1 returned=5  FP=[d20,d12,d41,d40]
- q47 `咖啡 AND 拿铁` [boolean] rel=1 returned=3  FP=[d36,d08]
- q48 `苹果 OR 咖啡` [boolean] rel=5 returned=2  FN=[d02,d08,d36]
- q51 `"中文" AND (测试 OR 检索) NOT 排除` [injection] rel=0 returned=6  FP=[d26,d09,d42,d04,d13,d11]

