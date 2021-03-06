# SlotBot: Hacking slot machines to win the jackpot using a hidden camera and brute-force search


So it turns out that there's a game on a specific brand of slot machine that's basically like an extreme version of Trivial Pursuit. It *also* turns out that the game ROM (containing all the answers) can be found online.

This was inspired by Claude Shannon (father of information theory) who figured out a way to gain a [probabilistic edge roulette wheels](http://nautil.us/issue/50/emergence/claude-shannon-the-las-vegas-cheat) with Edward Thorp.

Enjoy!

**Disclaimer: ROM legitimately acquired and not included for obvious reasons. This game was recently retired from machines.**

Here's the pipeline:

- Capture image of slot machine screen with buttonhole camera to raspberry pi
- Process image to undo perspective shift and segment into question and answer boxes with OpenCV
- Pass processed question boxes to Google Tesseract for text recognition
- Run OCR text through a hand-designed brute-force search to get the most likely answer
- Pass answer through text-to-speech engine and into hidden earpiece

### The game

The game asks you a series of general knowledge questions. It presents you with a choice of four answers, where one is correct. The more you get right, the more money you build up, until you win the jackpot.

### Decrypting the game files
The game data files look like [this](https://github.com/tensorman/slotbot/blob/master/jackpot_q_bank/UK_geography_01.QQQ); encrypted, unreadable text.
Fortunately, it turned out that they were encrypted using an [xor cipher](https://en.wikipedia.org/wiki/XOR_cipher).
This means that we can fairly easily [write a script](https://en.wikipedia.org/wiki/Chosen-plaintext_attack) to get a list of questions and answers in human-readable, decrypted form (run `python decrypt.py`).

### Designing a brute force search

Now we have the data, the fun begins. We need to read the screen, match what we can see to a question in the data bank via a brute force search, and read out the corresponding correct answer.

I initially tried to do this with the question data alone, ignoring the answers. Unfortunately, this doesn't work. Optical character recognition is imperfect, especially when running in real-time off a bad camera. About 30% of the characters will typically be misread. This means that the OCR-read question text is typically too garbled to identify which exact question it corresponds to; we can only narrow the search down to about 30 possible candidates.

So, we need to use the information provided by the answers to help identify which question we are looking at. This makes the brute force search a little more tricky, but still possible.

The two basic ingredients of this brute-force search will be (i) a way to compare two strings for similarity; and, using this, (ii) a metric to rank similarity between imperfectly observed question/answer pairs and true samples from the database.

We use the [Levenschtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) to define the similarity between two strings, defined as the minimum number of edits needed to change one string into another. Since a longer string tends to accumulate more reading errors, we'll normalize the Levenschtein distance over its length.

We form a confusion matrix of OCR answers against database answers. Taking the Frobenius inner product between this and every 4-D [permutation matrix](https://en.wikipedia.org/wiki/Permutation_matrix) will give us the metric we need. We can brute-force search over this to find the correct answer. The intuition behind this algorithm is that we're taking the dot product between the observed confusion matrix (which is noisy due to poor observability), and idealized confusion matrices (assuming perfect observability). The idealized confusion matrices take the form of permutation matrices because the four answers can appear in any permutation.

The code to carry out the brute force search can be found in `/src/`.

### Hardware

I bought a [raspberry pi 2](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/) to run the software, and used a TTS engine (Google Tesseract) to read out the answer into an earpiece.

I actually couldn't get the code to run fast enough on the raspberry pi to be useful (a single pass took about 30s). The bottleneck was opencv and tesseract (the only bits I couldn't optimize), so I ended up having to pipe the image over wifi to be processed by a laptop in a backpack. The code running on the rpi can be found in `./pi_interface.py`.
