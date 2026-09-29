For a document that large, command-line tools like `qpdf` or `pdftk` are the fastest and most memory-efficient approaches. They manipulate PDF page trees directly without re-rendering or decompressing entire pages, making them capable of processing 2,000 pages in just a few seconds.

  

### Method 1: Using `qpdf` (Recommended)

`qpdf` is fast, preserves bookmarks/metadata well, and supports precise page range slicing.

  

1. **Install `qpdf`:**
    
      
    
    Bash
    
    ```
    # Debian / Ubuntu
    sudo apt install qpdf
    
    # macOS (Homebrew)
    brew install qpdf
    ```
    
2. **Split into chunks of $N$ pages (e.g., 500 pages each):**
    
      
    
    Bash
    
    ```
    qpdf --split-pages=500 input.pdf output_chunk_%d.pdf
    ```
    
    _This outputs `output_chunk_1.pdf` (pages 1–500), `output_chunk_2.pdf` (501–1000), etc._
    
      
    
3. **Extract a specific range:**
    
      
    
    Bash
    
    ```
    qpdf input.pdf --pages . 1-500 -- output_part1.pdf
    ```
    
