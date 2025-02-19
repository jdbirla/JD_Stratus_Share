# Stratus learning

Here's how to create a `.tar.gz` archive with **multiple files** on Windows, using both the command line and 7-Zip:

---

### **Method 1: Command Prompt (Windows 10/11+)**
1. **Place all your files** (e.g., `file1.xsl`, `file2.xml`, `image.png`) in a folder (e.g., `C:\myfiles`).

2. **Open Command Prompt** and navigate to the folder:
   ```bash
   cd C:\myfiles
   ```

3. **Create the `.tar.gz` archive** with multiple files:
   - **Specific files**:
     ```bash
     tar -czvf myarchive.tar.gz file1.xsl file2.xml image.png
     ```
   - **All files in the folder** (wildcard `*`):
     ```bash
     tar -czvf myarchive.tar.gz *
     ```
   - **Specific file types** (e.g., all `.xsl` files):
     ```bash
     tar -czvf myarchive.tar.gz *.xsl
     ```

---

### **Method 2: 7-Zip (GUI)**
1. **Select multiple files** in File Explorer:
   - Hold **`Ctrl`** and click the files you want to include.

2. **Right-click** the selected files > **7-Zip** > **Add to archive...**.

3. **Configure the TAR archive**:
   - Set format to **`tar`**.
   - Name the archive (e.g., `myarchive.tar`).  
     ![7-Zip TAR creation](https://i.imgur.com/3e5Yjz4.png)

4. **Compress the TAR to `.tar.gz`**:
   - Right-click the new `myarchive.tar` > **7-Zip** > **Add to archive...**.
   - Set format to **`gzip`** and name it `myarchive.tar.gz`.

---

### **Verify the Archive**
Check if all files are included:
- **Command Prompt**:
  ```bash
  tar -tzvf myarchive.tar.gz
  ```
- **7-Zip**:  
  Right-click the `.tar.gz` file > **7-Zip** > **Open archive**.

---

### **Notes**
- **Spaces in filenames**: Enclose names in quotes in the command line:
  ```bash
  tar -czvf archive.tar.gz "file 1.xsl" "file 2.xml"
  ```
- **Subdirectories**: To include an entire folder (and its contents), specify the folder name:
  ```bash
  tar -czvf archive.tar.gz myfolder/
  ```

This method works for any combination of files (XSL, XML, images, etc.).
