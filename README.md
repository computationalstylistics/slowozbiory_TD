# Slowozbiory “Tekstów Drugich”

Repozytorium zawiera dane wejściowe oraz opis eksperymentów do studium na temat modelowania tematycznego czasopisma “Teksty Drugie”:

> Maryl, M. and Eder, M. (forthcoming). Słowozbiory “Tekstów Drugich”.


## Streszczenie

Artykuł prezentuje analizę problematyki poruszanej na łamach “Tekstów Drugich” w latach 1990–2012 z wykorzystaniem metody modelowania tematycznego. Na podstawie 1923 artykułów z tego okresu autorzy uzyskali i poddali interpretacji 120 wyłonionych automatycznie słowozbiorów, tj. tematów obecnych na łamach pisma. Wyrożniono tematy stałe, a także takie, którymi zainteresowanie rosło lub malało. Zastosowanie metod ilościowych pozwoliło ująć historię dyscypliny nie tyle jako serię odrębnych etapów, ile ciągły proces, w ramach którego współistnieją różne konfiguracje zagadnień. 

## Dane

Folder `DATA` zawiera dane wejściowe potrzebne do wytrenowania modelu tematycznego. W podfolderze `subset_lemmatized` znajdują się wszystkie teksty z czasopisma obejmujące lata 1990–2012, w wersji zlematyzowanej. Oprócz tego folder zawiera oryginalną tabelę frekwencji słów pozyskanych z surowych tekstów (plik `word_raw_frequencies.RData`), a także trzy wersje tabeli użyte w naszych trzech modelach A, B i C. Tabele te różnią się wyłącznie tym, że zostały z nich usunięte słowa ze stoplisty. Same stoplisty znaleźć można w folderze `STOPWORDS`, podczas gdy folder `MODELS` zawiera trzy końcowe modele tematyczne. W naszych eksperymentach korzystaliśmy wyłącznie z modelu `topic_model_k-120_no-NER.RData`.

## Kod

Dokument zapisany [tutaj](https://github.com/computationalstylistics/slowozbiory_TD/blob/main/kod_do_eksperymentow.md) zawiera opis eksperymentu wraz z wykonywalnym kodem.

