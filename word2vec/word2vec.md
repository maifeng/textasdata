
<style>
.reveal section p {
  color: black;
  font-size: .7em;
  font-family: 'Helvetica'; #this is the font/color of text in slides
}


.section .reveal .state-background {
    background: white;}
.section .reveal h1,
.section .reveal p {
    color: black;
    position: relative;
    top: 4%;}



</style>


Introduction to Word2Vec
========================================================
author: Chris Bail 
date: Duke University
autosize: true
transition: fade  
  website: https://www.chrisbail.net  
  github: https://github.com/cbail  
  Twitter: https://www.twitter.com/chris_bail

========================================================

# **What is word2vec?**

What is word2vec?
========================================================

<img src="word2vec_simple_vizual.png" height="400" />


What is word2vec?
========================================================

<img src="skip_gram_mikolov.png" height="400" />

========================================================

# **Using the Skip-Gram Model**

Installation
========================================================

We will be using the library `keras`

`keras` uses the TensorFlow backend, so installing this package requires a few more steps than the typical package on CRAN

The following code installs the `keras` package, which comes with a function that will install the TensorFlow dependencies


```r
install.packages("keras")
library(keras)
```

Installation
========================================================

To finish the installation process,


```r
install_keras()
```

Depending on what version of Python you have, this may give you an error. You may need to open your terminal and run the following lines of code:

*Note: If you have to run this code in the terminal, copying and pasting both at the same time will give you an error. You need to run them one by one, as the first line will prompt you to enter your computer user's password*
```
sudo /usr/bin/easy_install pip
sudo /usr/local/bin/pip install --upgrade virtualenv
```

Other packages we need for this example
========================================================

 (You may need to use `install.packages("package")` on some of these first)

```r
library(reticulate)
library(purrr)
library(text2vec) # note that this is a beta version of the package. code/names offuntions that comes from this package may change in future versions
library(dplyr)
library(Rtsne)
library(ggplot2)
library(plotly)
```


Preprocessing
========================================================


```r
load(url("https://cbail.github.io/Elected_Official_Tweets.Rdata"))
     
elected_no_retweets <- elected_official_tweets %>%
  filter(is_retweet == F) %>%
  select(c("screen_name", "text"))

tokenizer <- text_tokenizer(num_words = 20000)
tokenizer %>% fit_text_tokenizer(elected_no_retweets$text)
```

The Skip-Gram Model
========================================================
We need to define a function to yield batches of a certain size for model training:


```r
skipgrams_generator <- function(text, tokenizer, window_size, negative_samples) {
  gen <- texts_to_sequences_generator(tokenizer, sample(text))
  function() {
    skip <- generator_next(gen) %>%
      skipgrams(
        vocabulary_size = tokenizer$num_words, 
        window_size = window_size, 
        negative_samples = 1
      )
    x <- transpose(skip$couples) %>% map(. %>% unlist %>% as.matrix(ncol = 1))
    y <- skip$labels %>% as.matrix(ncol = 1)
    list(x, y)
  }
}
```

Defining the Keras Model
========================================================


```r
embedding_size <- 128  # Dimension of the embedding vector.
skip_window <- 5       # How many words to consider left and right.
num_sampled <- 1       # Number of negative examples to sample for each word.

input_target <- layer_input(shape = 1)
input_context <- layer_input(shape = 1)
```

Defining the Embedding Matrix
========================================================


```r
embedding <- layer_embedding(
  input_dim = tokenizer$num_words + 1, 
  output_dim = embedding_size, 
  input_length = 1, 
  name = "embedding"
)

target_vector <- input_target %>% 
  embedding() %>% 
  layer_flatten()

context_vector <- input_context %>%
  embedding() %>%
  layer_flatten()
```

Keras Model Cont.
========================================================


```r
dot_product <- layer_dot(list(target_vector, context_vector), axes = 1)
output <- layer_dense(dot_product, units = 1, activation = "sigmoid")

model <- keras_model(list(input_target, input_context), output)
model %>% compile(loss = "binary_crossentropy", optimizer = "adam")
summary(model)
```

Model Training
========================================================

You may have to play around with the number of steps and epochs you want to use


```r
model %>%
  fit_generator(
    skipgrams_generator(elected_no_retweets$text, tokenizer, skip_window, negative_samples), 
    steps_per_epoch = 1000, epochs = 2
  )
```

Extract the Embeddings
========================================================


```r
embedding_matrix <- get_weights(model)[[1]]

words <- data_frame(
  word = names(tokenizer$word_index), 
  id = as.integer(unlist(tokenizer$word_index))
)

words <- words %>%
  filter(id <= tokenizer$num_words) %>%
  arrange(id)

row.names(embedding_matrix) <- c("UNK", words$word)
```

Explore Embeddings
========================================================


```r
find_similar_words <- function(word, embedding_matrix, n = 5) {
  similarities <- embedding_matrix[word, , drop = FALSE] %>%
    sim2(embedding_matrix, y = ., method = "cosine")
  
  similarities[,1] %>% sort(decreasing = TRUE) %>% head(n)
}

find_similar_words("republican", embedding_matrix)
find_similar_words("democrat", embedding_matrix)
find_similar_words("partisan", embedding_matrix)
```
As you can see, we might need to go back and clean up the text a bit more before tokenizing (getting rid of URLs and pictures)

Visualize Embeddings
========================================================


```r
tsne <- Rtsne(embedding_matrix[2:500,], perplexity = 50, pca = FALSE)

tsne_plot <- tsne$Y %>%
  as.data.frame() %>%
  mutate(word = row.names(embedding_matrix)[2:500]) %>%
  ggplot(aes(x = V1, y = V2, label = word)) + 
  geom_text(size = 3)
tsne_plot
```

Visualize Embeddings
========================================================

<img src="Tweet_embeddings.png" height="400" />
