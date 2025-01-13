---
layout: post
title: "Saving an image to MacOS Photo album"
date: 2025-01-12
category: Technical Review
tags:
  - macOS
  - Swift
  - Automation
  - Photo Management
  - Command Line Tools
author: Danny O'Brien <danny@spesh.com>
---

# Saving an image to the MacOS Photo Album

The Photo album on Mac desktops occupies an odd place in the file-system
hierarchy. You're encouraged to access it only through the Photo app, but it
also appears as a location when you're loading images in some application.

What you can't do -- at least very easily -- is save an image to it. Safari has
a right-menu option to save browser images to Photos, but I don't use Safari. I
do use Photos as a dumping ground for images, partly because it's a much more
convenient place for saving and editing them on iOS devices.

What I've wanted, for a while, is a command to take whatever image is in the
clipboard (I'm a big clipboard user), and save that out as a photo. I asked ChatGPT to [write me a program to do this](https://chatgpt.com/share/67847158-8648-800a-af48-ee9b946c3e03), and it came up trumps. Here's our conversation, with code.

You: 

I'd like to write a function or extension or something that would let me "save clipboard (image)  as a Photo" -- ie save any image data as a new photo in Apple's Photo album -- in as many plaes in the macos interface as possible. Raycast or the command line might be the best place for this function to work, but I'm not wed to them. It should be a small, self-contained program, with fre dependencies. Language is unimportant, but Swift or AppleScript might be the best choices.

ChatGPT: 

Thought about saving clipboard image to Photos for 13 seconds
Below is a self‐contained approach in Swift that:

1. Reads the current clipboard as an image (if any).

2. Saves it to a temporary file.

3. Tells Photos.app to import that file into your library.
This is all done with **no third‐party dependencies** , just Swift + AppleScript calls under the hood. You can compile it into a small command‐line tool (or just run it as a Swift “script”) and then trigger it from anywhere: a Terminal alias, Raycast, FastScripts, etc.

---


## 1. The Swift source 
Create a file named `ClipboardToPhotos.swift` with these contents:

```swift
#!/usr/bin/env swift

import Cocoa
import Foundation

// 1. Get the general pasteboard
let pasteboard = NSPasteboard.general

// 2. Try to read an NSImage from the clipboard
guard let image = NSImage(pasteboard: pasteboard) else {
    fputs("No image found on the clipboard.\n", stderr)
    exit(1)
}

// 3. Convert the image to PNG data
guard
    let tiffData = image.tiffRepresentation,
    let bitmap = NSBitmapImageRep(data: tiffData),
    let pngData = bitmap.representation(using: .png, properties: [:])
else {
    fputs("Failed to convert image to PNG.\n", stderr)
    exit(1)
}

// 4. Write the PNG data to a temporary location
let tempDir = URL(fileURLWithPath: NSTemporaryDirectory())
let tempFileURL = tempDir.appendingPathComponent("clipboard_image.png")
do {
    try pngData.write(to: tempFileURL, options: .atomic)
} catch {
    fputs("Failed to write PNG to disk: \(error)\n", stderr)
    exit(1)
}

// 5. Use AppleScript to tell Photos to import that file
let scriptSource = """
    tell application "Photos"
        import POSIX file "\(tempFileURL.path)"
    end tell
    """

if let script = NSAppleScript(source: scriptSource) {
    var errorDict: NSDictionary?
    script.executeAndReturnError(&errorDict)

    if let err = errorDict {
        fputs("AppleScript error: \(err)\n", stderr)
        exit(1)
    } else {
        print("Successfully imported image into Photos.")
    }
} else {
    fputs("Failed to create AppleScript object.\n", stderr)
    exit(1)
}
```

### How it works 
 
- **Step 2**  uses `NSImage(pasteboard:)` to pull whatever image is on the clipboard.
 
- **Step 3–4**  convert it to PNG and write it to a temporary file.
 
- **Step 5**  uses AppleScript to tell the Photos app to import that newly‐created file.


---


## 2. Running / Installing 

### A) As a Swift “script” 
You can run it directly (assuming you saved it as `ClipboardToPhotos.swift` and marked it executable):

```bash
chmod +x ClipboardToPhotos.swift
./ClipboardToPhotos.swift
```

If your clipboard contains an image, it will appear in Photos shortly thereafter. If no image is found, it prints an error and exits.

### B) As a compiled command‐line tool 
Compile it once and then place the binary anywhere in your `$PATH` (e.g., in `/usr/local/bin` if you like):

```bash
swiftc ClipboardToPhotos.swift -o ClipboardToPhotos
mv ClipboardToPhotos /usr/local/bin
```
Then you can just type `ClipboardToPhotos` anywhere (Terminal, scripts, Raycast, etc.) to run it.

---


## 3. Integrating with Raycast or other launchers 
 
- **Raycast** : Create a script command pointing to your compiled tool. Then you can trigger it via a hotkey, a quick search, or any Raycast action.
 
- **FastScripts** : You can drop either the script file or the compiled tool in a script folder and assign a hotkey.
 
- **Automator / Shortcuts** : You could also wrap this command in an Automator “Run Shell Script” or a Shortcuts action if you want a more GUI approach.


---


### Notes and tips 
 
- **No third‐party tools** : This avoids tools like `pngpaste` from Homebrew, which won’t be available by default on a fresh macOS install.
 
- **Clipboard formats** : If the clipboard holds something that isn’t an `NSImage`, the script will gracefully abort.
 
- **Error messages** : If something goes wrong (no image, can’t convert, AppleScript error, etc.), you’ll see diagnostic output in the terminal (or wherever you capture stderr).
 
- **File type** : The example uses PNG, but you could use JPEG or another format by changing the `representation(using: .png, properties: [:])` line.

This approach should be minimal, free of external dependencies, and easy to tie into any workflow you prefer. Enjoy saving your clipboard images straight into Photos!






