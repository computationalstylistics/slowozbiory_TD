# Kod użyty w eksperymentach


Poniższy kod jest całokowice replikowalny w tym sensie, że każdy etap naszego postępowania badawczego i każdy eksperyment przez nas przeprowadzony można powtórzyć. Poniższy kod składa się jednak z niezależnych elementów; użycie każdego z nich wymaga ustawienia ścieżek dostępu do plików (lub do korpusu). Nie jesteśmy w stanie przewidzieć, w którym miejscu swojego dysku czytelnik niniejszej instrukcji umieści swoje pliki, dlatego zostawiliśmy w kodzie oryginalne ścieżki dostępu.




## Selekcja tekstów

Uzgodnienie pliku z metadanymi z plikami w korpusie (wydzielenie podzbioru):

``` R
a = read.csv("/Users/m/Downloads/TD_MapowanieDanych_biblio_txt.xlsx - Mapowanie.csv")[,1]
setwd("/Users/m/Desktop/TXT1990-2014uzup_zlem")
b = list.files()
new.folder = "subset_lemmatized"
list.of.files = b[(b %in% a)] 
file.copy(list.of.files, new.folder)
```

W wyniku tej operacji powinniśmy dostać 1870 tekstów wyselekcjonowanych z całego zbioru, zapisanych w folderze `subset_lemmatized`.







## Przygotowanie tabeli frekwencji

Folder `DATA/subset_lemmatized` zawiera 1870 plików z artykułami z Tekstów Drugich. Ładujemy pliki i tworzymy listę frekwencyjną (uwaga: na potrzeby TM nie będą to względne frekwencje, lecz surowe wystąpienia poszczególnych słów, a zatem liczby naturalne zamiast ułamków):


``` R
library(stylo)

parsed.corpus = load.corpus.and.parse(files = dir(), corpus.dir = "DATA/subset_lemmatized", corpus.lang = "Polish", encoding = "UTF-8")

freqlist = make.frequency.list(parsed.corpus, value = TRUE, relative = FALSE)
# trim the list at the freqnency >2
freqlist = names(freqlist)[freqlist > 2]
frequencies = make.table.of.frequencies(parsed.corpus, freqlist, relative = FALSE)

save(frequencies, file = "word_raw_frequencies.RData")
```

Końcowa tabela frekwencji zostaje zapisana do pliku `word_raw_frequencies.RData` i nie będzie już więcej zmieniana – nie ma więc potrzeby odwoływania się do poszczególnych plików z tekstami.






## Usunięcie słów ze stoplisty


Trzy podejścia: zastosowanie wag TFIDF do wszystkich słów i usunięcie tych, które mają średnią wagę >1 (czyli występujących powszechnie) oraz <0.05 (a zatem bardzo rzadkich).

Do tej listy ręcznie dodane pojedyncze litery, konwencjonalne skróty, kilka wyrazów obcojęzycznych etc.


``` R
################### stopwords based on TFIDF scoring
load("word_raw_frequencies.RData")

copy_frequencies = frequencies
copy_frequencies[copy_frequencies > 0] = 1
df = colSums(copy_frequencies)
no_of_docs = dim(copy_frequencies)[1]
idf = log(no_of_docs / df)
# TF/IDF
tfidf = t( t(frequencies) * idf )

# avg tfidf
av_tfidf = colSums(tfidf) / no_of_docs
av_tfidf = sort(av_tfidf)


hipass_filter = names(av_tfidf[av_tfidf <0.05])
lowpass_filter = names(av_tfidf[av_tfidf >1])

top_100_words = c()  # colnames(frequencies)[1:100]

add_words = c("a", "ą", "b", "c", "ć", "d", "e", "f", "g", "h", "i", "j", "k", "l", "ł", "m", "n", "ń", "o", "ó", "p", "q", "r", "s", "ś", "t", "u", "v", "x", "y", "z", "ż", "ź",
"nych", "wy", "prze", "ki", "nia", "przy", "tyczny", "dera", "żc", "ha", "ry", "ego",
"gram", "litr", "sekunda", "bit",
"zaś", "przecież", "trzeba", "natomiast", "zwłaszcza", "również", "bez", "jeśli", "albo", "bo", "były", "niż", 
"tłum", "oprac", "wyd", "op", "cit", "st", "cz", "sł",
"university", "press", "new", "york", "london", "paris", "ed", "the", "of", "in", "und", "die", "von", "das", "de", "la", "le", "les", "du", "zob", "tamże")
stopwords = sort(unique(c(lowpass_filter, hipass_filter, top_100_words, add_words)))

words_OK = setdiff(colnames(frequencies), stopwords)
filter_words = colnames(frequencies) %in% words_OK

dtm_pruned = frequencies[ , filter_words]

save(dtm_pruned, file = "dtm_pruned_TFIDF.RData")
```

Tabela z usuniętym słowami ze stoplisty zapisana w pliku `dtm_pruned_TFIDF.RData`. Być może należy rozważyć usunięcie również kilkudziesięciu najczęstszych słów, choć to akurat jest celem kolejnego podejścia:


``` R
################### stopwords based on frequencies

# skip the 100 top words, and the words that occurred less than 5 times
filter_words = 101 : sum(as.numeric(colSums(frequencies) > 5))

dtm_pruned = frequencies[ , filter_words]

save(dtm_pruned, file = "dtm_pruned_100MFWs-culled.RData")
```

Końcowa tabela zapisana w pliku `dtm_pruned_100MFWs-culled.RData`. Wreszcie trzecie podejście: bez stoplisty, usunięte wyłącznie rzadkie słowa (<5 wystąpień), żeby sprawdzić jakość modelu, gdy nie dba się o cyzelowanie stoplisty. Niby wynik łatwo przewidzieć, ale ciekawe może być porównanie wartości perplexity i loglikelihood dla trzech modeli wykorzystujących różne podejścia do stoplisty.


``` R
# skip the words that occurred less than 5 times
filter_words = 1 : sum(as.numeric(colSums(frequencies) > 5))

dtm_pruned = frequencies[ , filter_words]
save(dtm_pruned, file = "dtm_pruned_no-stoplist.RData")
```





## Optymalna liczba topików 

Wykonanie poniższego kodu zajmuje kilkadziesiąt godzin na mocnym serwerze: nie sprawdzajcie tego w domu! Na serwerze musi działać pakiet `ldatuning`, trzeba też przekopiować trzy tabele z frekwencjami uzyskanymi powyżej i zapisanymi do plików.



``` R
library(ldatuning)

# first run: stopwords based on TFIDF weighting

load("dtm_pruned_TFIDF.RData")

results = FindTopicsNumber(
  dtm_pruned,
  topics = seq(from = 10, to = 300, by = 10),
  metrics = c("Griffiths2004", "CaoJuan2009", "Arun2010", "Deveaud2014"),
  method = "Gibbs",
  control = list(seed = 1234),
  #mc.cores = 2L,
  verbose = TRUE
)

save(results, file = "TD_k_optimize_for_10-300_topics_TFIDF.RData")


# second run: stopwords based on word frequencies

load("dtm_pruned_100MFWs-culled.RData")

results = FindTopicsNumber(
  dtm_pruned,
  topics = seq(from = 10, to = 300, by = 10),
  metrics = c("Griffiths2004", "CaoJuan2009", "Arun2010", "Deveaud2014"),
  method = "Gibbs",
  control = list(seed = 1234),
  #mc.cores = 2L,
  verbose = TRUE
)

save(results, file = "TD_k_optimize_for_10-300_topics_100MFWs.RData")



# third run: no stopwords removed

load("dtm_pruned_no-stoplist.RData")

results = FindTopicsNumber(
  dtm_pruned,
  topics = seq(from = 10, to = 300, by = 10),
  metrics = c("Griffiths2004", "CaoJuan2009", "Arun2010", "Deveaud2014"),
  method = "Gibbs",
  control = list(seed = 1234),
  #mc.cores = 2L,
  verbose = TRUE
)

save(results, file = "TD_k_optimize_for_10-300_topics_no-stoplist.RData")

```


Po wykonaniu powyższego kodu trzy wynikowe pliki `.RData` należy skopiować do lokalnego komputera i wykonać np. poniższy kod:


``` R
library(ldatuning)

load("TD_k_optimize_for_10-300_topics_TFIDF.RData")
FindTopicsNumber_plot(results)

load("TD_k_optimize_for_10-300_topics_100MFWs.RData")
FindTopicsNumber_plot(results)

load("TD_k_optimize_for_10-300_topics_no-stoplist.RData")
FindTopicsNumber_plot(results)

```





## Modelowanie tematyczne

Choć pierwotnie testowane przez nas modele były wygenerowane przez pakiet `mallet` z poziomu R, dla konsekwencji metodologicznej warto się trzymać jednego rozwiązania we wszystkich eksperymentach, a skoro użyty powyżej pakiet `ldatuning` opiera się na pakiecie `topicmodels`, wszystkie pozostałe obliczenia również wykonaliśmy za pomocą tego pakietu.

Potrzebny będzie pakiet `topicmodels`

``` R
library(topicmodels)
```

Typowe użycie: model LDA, Gibbs sampling, 120 słowozbiorów (domyślnie 2000 iteracji):

``` R
topic_model = LDA(dtm_pruned, k = 120, method = "Gibbs", control = list(seed = 1234, verbose = 1))
```

Inne rozłożenie iteracji:

``` R
load("dtm_pruned_TFIDF.RData")

topic_model = LDA(dtm_pruned, k = 120, method = "Gibbs", control = list(seed = 1234, burnin = 1000, thin = 100, iter = 1000, verbose = 1))

save(topic_model, file = "topic_model_k-120_tfidf.RData")
```

Model zapisany do pliku `topic_model_k-120_tfidf.RData`. Jako podstawa wzięta tabela z usuniętymi słowami o dużej i bardzo małej wartości TFIDF (zob. wyżej).






## Trzy modele


Poniżej kod, który użyliśmy do wytrenowania trzech modeli tematycznych -- wariantu A, B i C. Wszystkie zostały zapisane do plików, odpowiednio `topic_model_k-120_tfidf.RData`, `topic_model_k-120_no-names.RData` oraz `topic_model_k-120_no-NER.RData`. Pliki te dostępne są w niniejszym repozytorium w folderze `MODELS`. W części eksploracyjno-interpretacyjnej wykorzystujemy wyłącznie model C, a więc ostatni z wyżej wymienionych plików.


1. model A, standardowy


``` R
load("dtm_pruned_TFIDF.RData")
topic_model = LDA(dtm_pruned, k = 120, method = "Gibbs", control = list(seed = 1234, burnin = 1000, thin = 100, iter = 1000, verbose = 1))
save(topic_model, file = "topic_model_k-120_tfidf.RData")
```



2. model B, z usuniętymi nazwiskami z bibliografii

``` R
load("dtm_pruned_TFIDF.RData")
# plik "nazwiska.txt" wzięty z dysku google'a
nazwiska_raw = readLines("nazwiska.txt")
nazwiska = unlist(strsplit(nazwiska_raw, "[ ,.-]+"))
nazwiska = unique(tolower(nazwiska))
filter_words = !(colnames(dtm_pruned) %in% nazwiska)
dtm_no_names = dtm_pruned[ , filter_words]
# takie oto nazwiska zostaną usunięte z modelu:
colnames(dtm_pruned)[colnames(dtm_pruned) %in% nazwiska]
#
topic_model = LDA(dtm_no_names, k = 120, method = "Gibbs", control = list(seed = 1234, burnin = 1000, thin = 100, iter = 1000, verbose = 1))
save(topic_model, file = "topic_model_k-120_no-names.RData")
```




3. model C, z usuniętymi nazwiskami z NER-a


``` R
load("dtm_pruned_TFIDF.RData")
# plik "nazwiska.txt" wzięty z dysku google'a
ner_raw = read.csv("NER_do_stoplisty.xlsx - NERdoStoplisty.csv")
# bierzemy tylko >4 wystąpienia (czyli 5+) według danych z 3. kolumny:
ner_trimmed = ner_raw[ner_raw[,3] > 4, ]
# a teraz bierzemy tylko 2. kolumnę (tj. lematy)
nazwiska_raw = ner_trimmed[,2]
#
nazwiska = unlist(strsplit(nazwiska_raw, "[ ,.-]+"))
nazwiska = unique(tolower(nazwiska))
filter_words = !(colnames(dtm_pruned) %in% nazwiska)
dtm_no_ner = dtm_pruned[ , filter_words]
# takie oto nazwiska zostaną usunięte z modelu:
colnames(dtm_pruned)[colnames(dtm_pruned) %in% nazwiska]
#
save(dtm_no_ner, file = "dtm_pruned_no-NER.RData")
#
load("dtm_pruned_no-NER.RData")
topic_model = LDA(dtm_no_ner, k = 120, method = "Gibbs", control = list(seed = 1234, burnin = 1000, thin = 100, iter = 1000, verbose = 1))
save(topic_model, file = "topic_model_k-120_no-NER.RData")
```

W dalszej eksploracji modeli nie trzeba ich będzie już trenować, nie trzeba nawet wracać do danych wejściowych. Zapisany plik modelu od tej chwili jest wszystkim, co będziemy potrzebować w dalszej eksploracji.



## Eksploracja słowozbiorów

Pakiet `topicmodels` wprawdzie został już powyżej załadowany, ale gdyby rzecz wykonywać od zera, trzeba by jeszcze raz go uaktywnić, przyda się też wczytać model z pliku:

``` R
library(topicmodels)
load("topic_model_k-120_no-NER.RData")
```
Można wtedy zobaczyć, jakie 10 najważniejszych słów składa się na każdy słowozbiór:

``` R
# to get 10 top words in each topic
terms(topic_model, 10)
```


Dostęp do wag poszczególnych słów w poszczególnych słowozbiorach jest nieco bardziej skomplikowany, ale można tę informację wydobyć przez sięgnięcie po funkcję `posterior`:


``` R
model_weights = posterior(topic_model)
topic_words = model_weights$terms
docs_topics = model_weights$topics
```

Teraz sprawa jest prosta. Korzystając z biblioteki `wordcloud`, można przedsawić np. 50 najważniejszych słów np. piątym słowozbiorze:

``` R
library(wordcloud)
topic_id = 5
no_of_words = 50
top_words_with_weights = sort(topic_words[topic_id,], decreasing = T)[1:no_of_words]
wordcloud(names(top_words_with_weights), top_words_with_weights, random.order = FALSE, rot.per = 0)
```

Jest rzeczą oczywistą, że skoro można zrobić chmurę słów dla jednego słowozbioru, można też przedstawić je wszystkie naraz i zapisać do poszczególnych plików PNG (uwaga: poniższy kod generuje 120 plików w bieżącym katalogu!):

``` R
no_of_words = 50
no_of_topics = topic_model@k

model_weights = posterior(topic_model)
topic_words = model_weights$terms
docs_topics = model_weights$topics

for(i in 1 : no_of_topics) {
	topic_id = i
	current_topic = sort(topic_words[topic_id,], decreasing = T)[1:no_of_words]
	png(file = paste("topic_", topic_id, ".png", sep=""))
	wordcloud(names(current_topic), current_topic, random.order = FALSE, rot.per = 0)
	dev.off()
}
```


Udział słowozbiorów w dowolnym dokumencie z korpusu, np. H. Markiewicza z rocznika 2000 nr 3 (672. dokument w korpusie):


``` R
doc_number = 672
plot(docs_topics[doc_number,], type = "h", lwd = 3, col = rgb(1,0,0,0.5), main = rownames(docs_topics)[doc_number], xlab = "słowozbiór", ylab = "proporcja")
```

A tak R. Nycz z rocznika 2012 nr 3:

``` R
doc_number = 1804
plot(docs_topics[doc_number,], type = "h", lwd = 3, col = rgb(1,0,0,0.5), main = rownames(docs_topics)[doc_number], xlab = "słowozbiór", ylab = "proporcja")
```

Oczywiście listę wszystkich dokumentów łatwo wydobyć stąd:

``` R
rownames(docs_topics)
```




## Słowozbiory chronologicznie


A jak by się rozkłdał słowozbiór, powiedzmy, 9 w funkcji czasu? Weźmy wszystkie artykuły z roku X i oszacujmy średnią wagę danego słowozbioru, i tak dla wszystkich lat. Najpierw załadujmy potrzebne funkcje i zmienne:

``` R
library(topicmodels)
library(mgcv)
load("topic_model_k-120_no-NER.RData")
model_weights = posterior(topic_model)
topic_words = model_weights$terms
docs_topics = model_weights$topics
```

A teraz właściwy kod (warto zwrócić uwagę, że linia trendu do model GAM wytrenowany na frekwencjach danego słowozbioru w funkcji czasu):

``` R
topic_ID = 9
year = gsub("^([0-9]{4}).*", "\\1", rownames(docs_topics))
current_topic = c()
for(i in unique(year) ) {
    topic_in_a_year = mean( docs_topics[year == i,topic_ID] )
    current_topic = c(current_topic, topic_in_a_year)
}
names(current_topic) = unique(year)

plot(current_topic ~ unique(year), main = paste("słowozbiór", topic_ID), xlab = "rok", ylab = "średni udział słowozbioru")
# fit a GAM model:
model_gam = gam(current_topic ~ s(as.numeric(unique(year)), bs = "cr"))
# add a trendline (the fitted model)
lines(model_gam$fitted.values ~ as.numeric(unique(year)), col = rgb(1, 0, 0, 0.6), lwd = 3)
```


Tutaj z kolei kod pozwalający obejrzeć wszystkie słowozbiory po kolei w ich rozwoju czasowym:


``` R

for(topic_ID in 1:120) {
    year = gsub("^([0-9]{4}).*", "\\1", rownames(docs_topics))
    current_topic = c()
    for(i in unique(year) ) {
        topic_in_a_year = mean( docs_topics[year == i,topic_ID] )
        current_topic = c(current_topic, topic_in_a_year)
    }
    names(current_topic) = unique(year)

    top_3_words = names(sort(topic_words[topic_ID,], decreasing = T)[1:3])
    plot_title = paste(top_3_words, collapse = " | ")
    
    plot(current_topic ~ unique(year), main = paste(plot_title, "  (", topic_ID, ")", sep = ""), xlab = "rok",  ylab = "średni udział słowozbioru")
    model_gam = gam(current_topic ~ s(as.numeric(unique(year)), bs = "cr"))
    # add a trendline (the fitted model)
    lines(model_gam$fitted.values ~ as.numeric(unique(year)), col = rgb(1, 0, 0, 0.6), lwd = 3)    
    Sys.sleep(2)
}

```



