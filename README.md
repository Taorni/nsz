## NSZ
NSZ files are not a real format, they are functionally identical to NSP files. Their sole purpose to alert the user that it contains compressed NCZ files. NCZ files can be mixed with NCA files in the same container.

NSC_Builder supports compressing NSP to NSZ, and decompressing NSZ to NSP. The sample scripts located here are just examples of how the format works.

NSC_Builder can be downloaded at https://github.com/julesontheroad/NSC_BUILDER

## XCZ
XCZ files are not a real format, they are functionally identical to XCI files. Their sole purpose to alert the user that it contains compressed NCZ files. NCZ files can be mixed with NCA files in the same container.

## NCZ
These are compressed NCA files. The NCA's are decrypted, and then compressed using zStandard. Only NCA's with a 0x4000 byte header are supported (CNMT nca's are not supported).

The first 0x4000 bytes of a NCZ file is exactly the same as the original NCA (and still encrypted).

At 0x4000, there is the variable sized NCZ Header. It contains a list of sections which tell the decompressor how to re-encrypt the NCA data after decompression. It can also contain an optional block compression header allowing random read access.

All of the information in the header can be derived from the original NCA + Ticket, however it is provided preparsed to make decompression as easy as possible for third parties.

Directly after the NCZ header, the zStandard stream begins and ends at EOF. The stream is decompressed to offset 0x4000. If block compression is used the stream is splatted into independent blocks and can be decompressed as shown in https://github.com/nicoboss/nsz/blob/master/nsz/BlockDecompressorReader.py

```python
class Section:
	def __init__(self, f):
		self.magic = f.read(8) # b'NCZSECTN'
		self.offset = f.readInt64()
		self.size = f.readInt64()
		self.cryptoType = f.readInt64()
		f.readInt64() # padding
		self.cryptoKey = f.read(16)
		self.cryptoCounter = f.read(16)

class Block:
	def __init__(self, f):
		self.magic = f.read(8) # b'NCZBLOCK'
		self.version = f.readInt8()
		self.type = f.readInt8()
		self.unused = f.readInt8()
		self.blockSizeExponent = f.readInt8()
		self.numberOfBlocks = f.readInt32()
		self.decompressedSize = f.readInt64()
		self.compressedBlockSizeList = []
		for i in range(self.numberOfBlocks):
			self.compressedBlockSizeList.append(f.readInt32())

nspf.seek(0x4000)
sectionCount = nspf.readInt64()
for i in range(sectionCount):
	sections.append(Section(nspf))

if blockCompression:
	BlockHeader = Block(nspf)
```


## Compressor script

Requires hactool compatible keys.txt to be present with nsz.py. Only currently works with base games, updates, and DLC.

example usage:
nsz.py --level 18 -C title1.nsp title2.nsp title3.nsp

will generate title1.nsz title2.nsz title3.nsz

## Python requirements

py -3 -m pip install -r requirements.txt

## Usage
```
nsz.py --help
usage: nsz.py [-h] [-C] [-D] [-l LEVEL] [-B] [-s BS] [-V] [-t THREADS]
              [-o OUTPUT] [-w] [-i INFO] [--depth DEPTH]
              [-x EXTRACT [EXTRACT ...]] [-c CREATE]
              [file [file ...]]

positional arguments:
  file

optional arguments:
  -h, --help            show this help message and exit
  -C                    Compress NSP
  -D                    Decompress NSZ
  -l LEVEL, --level LEVEL
                        Compression Level
  -B, --block           Uses highly multithreaded block compression with
                        random read access allowing compressed games to be
                        played without decompression in the future however
                        this comes with a low compression ratio cost. Current
                        title installers do not support this yet.
  -s BS, --bs BS        Block Size for random read access 2^x while x between
                        14 and 32. Default is 20 => 1 MB. Current title
                        installers do not support this yet.
  -V, --verify          Verifies files after compression raising an unhandled
                        exception on hash mismatch and verify existing NSP and
                        NSZ files when given as parameter
  -t THREADS, --threads THREADS
                        Number of threads to compress with. Usless without
                        enabeling block compression using -B. Numbers < 1
                        corresponds to the number of logical CPU cores.
  -o OUTPUT, --output OUTPUT
                        Directory to save the output NSZ files
  -w, --overwrite       Continues even if there already is a file with the
                        same name or title id inside the output directory
  -i INFO, --info INFO  Show info about title or file
  --depth DEPTH         Max depth for file info and extraction
  -x EXTRACT [EXTRACT ...], --extract EXTRACT [EXTRACT ...]
                        extract / unpack a NSP
  -c CREATE, --create CREATE
                        create / pack a NSP
```

## Credits

SciresM for his hardware crypto functions; the blazing install speeds (50 MB/sec +) achieved here would not be possible without this.

