Do exactly these three things, in order, then stop.

1. Read the file `spike/target.txt`. It contains a random nonce line that you
   cannot know without reading it.
2. Append exactly one new line to `spike/target.txt` containing only: SPIKE-OK
3. Run this shell command:

       sha256sum spike/target.txt | cut -c1-12

   Write its 12-character output, and nothing else, into a new file
   `spike/proof.txt`.

Do not modify any other file. Do not create any other file.
