# API examples

This wiki provide simple example how to use tesseract-ocr API (v3.02.02) in C++.
It is expected that tesseract-ocr is correctly installed including all dependecies.
It is expected that user is familiar with C++, compiling and linking program on his platform, [though basic compilation examples are included for beginners with Linux](#compiling-c-api-programs-on-linux).

More details about tesseract-ocr API can found at [baseapi.h](https://github.com/tesseract-ocr/tesseract/blob/master/api/baseapi.h)



# Basic example

Code:

```c++
#include <tesseract/baseapi.h>
#include <leptonica/allheaders.h>

int main()
{
    char *outText;

    tesseract::TessBaseAPI *api = new tesseract::TessBaseAPI();
    // Initialize tesseract-ocr with English, without specifying tessdata path
    if (api->Init(NULL, "eng")) {
        fprintf(stderr, "Could not initialize tesseract.\n");
        exit(1);
    }

    // Open input image with leptonica library
    Pix *image = pixRead("/usr/src/tesseract/testing/phototest.tif");
    api->SetImage(image);
    // Get OCR result
    outText = api->GetUTF8Text();
    printf("OCR output:\n%s", outText);

    // Destroy used object and release memory
    api->End();
    delete [] outText;
    pixDestroy(&image);

    return 0;
}
```

Program must be linked to tesseract-ocr library and leptonica library.

If you want to restrict recognition to a sub-rectangle of the image - call _SetRectangle(left, top, width, height)_ after SetImage. Each SetRectangle clears the recogntion results so multiple rectangles can be recognized with the same image. E.g.
```c++
  api->SetRectangle(30, 86, 590, 100);
```


# GetComponentImages example

```c++
  Pix *image = pixRead("/usr/src/tesseract/testing/phototest.tif");
  tesseract::TessBaseAPI *api = new tesseract::TessBaseAPI();
  api->Init(NULL, "eng");
  api->SetImage(image);
  Boxa* boxes = api->GetComponentImages(tesseract::RIL_TEXTLINE, true, NULL, NULL);
  printf("Found %d textline image components.\n", boxes->n);
  for (int i = 0; i < boxes->n; i++) {
    BOX* box = boxaGetBox(boxes, i, L_CLONE);
    api->SetRectangle(box->x, box->y, box->w, box->h);
    char* ocrResult = api->GetUTF8Text();
    int conf = api->MeanTextConf();
    fprintf(stdout, "Box[%d]: x=%d, y=%d, w=%d, h=%d, confidence: %d, text: %s",
                    i, box->x, box->y, box->w, box->h, conf, ocrResult);
  }
```

# Result iterator example

There is possibility to get confidence value and BoundingBox per word from ResultIterator:

```c++
  Pix *image = pixRead("/usr/src/tesseract/testing/phototest.tif");
  tesseract::TessBaseAPI *api = new tesseract::TessBaseAPI();
  api->Init(NULL, "eng");
  api->SetImage(image);
  api->Recognize(0);
  tesseract::ResultIterator* ri = api->GetIterator();
  tesseract::PageIteratorLevel level = tesseract::RIL_WORD;
  if (ri != 0) {
    do {
      const char* word = ri->GetUTF8Text(level);
      float conf = ri->Confidence(level);
      int x1, y1, x2, y2;
      ri->BoundingBox(level, &x1, &y1, &x2, &y2);
      printf("word: '%s';  \tconf: %.2f; BoundingBox: %d,%d,%d,%d;\n",
               word, conf, x1, y1, x2, y2);
      delete[] word;
    } while (ri->Next(level));
  }
```

Of course there is possibility to use other [PageiteratorLevel](https://github.com/tesseract-ocr/tesseract/blob/master/ccstruct/publictypes.h#L204).



# Orientation and script detection (OSD) example

```c++
  const char* inputfile = "/usr/src/tesseract/testing/eurotext.tif";
  tesseract::Orientation orientation;
  tesseract::WritingDirection direction;
  tesseract::TextlineOrder order;
  float deskew_angle;

  PIX *image = pixRead(inputfile);
  tesseract::TessBaseAPI *api = new tesseract::TessBaseAPI();
  api->Init("/usr/src/tesseract/", "eng");
  api->SetPageSegMode(tesseract::PSM_AUTO_OSD);
  api->SetImage(image);
  api->Recognize(0);

  tesseract::PageIterator* it =  api->AnalyseLayout();
  it->Orientation(&orientation, &direction, &order, &deskew_angle);
  printf("Orientation: %d;\nWritingDirection: %d\nTextlineOrder: %d\n" \
         "Deskew angle: %.4f\n",
         orientation, direction, order, deskew_angle);
```

Explanation for result codes are in [publictypes.h](https://github.com/tesseract-ocr/tesseract/blob/master/ccstruct/publictypes.h#L86)


# Example of iterator over the classifier choices for a single symbol

```c++

  Pix *image = pixRead("/usr/src/tesseract/testing/phototest.tif");
  tesseract::TessBaseAPI *api = new tesseract::TessBaseAPI();
  api->Init(NULL, "eng");
  api->SetImage(image);
  api->SetVariable("save_blob_choices", "T");
  api->SetRectangle(37, 228, 548, 31);
  api->Recognize(NULL);

  tesseract::ResultIterator* ri = api->GetIterator();
  tesseract::PageIteratorLevel level = tesseract::RIL_SYMBOL;
  if(ri != 0) {
      do {
          const char* symbol = ri->GetUTF8Text(level);
          float conf = ri->Confidence(level);
          if(symbol != 0) {
              printf("symbol %s, conf: %f", symbol, conf);
              bool indent = false;
              tesseract::ChoiceIterator ci(*ri);
              do {
                  if (indent) printf("\t\t ");
                  printf("\t- ");
                  const char* choice = ci.GetUTF8Text();
                  printf("%s conf: %f\n", choice, ci.Confidence());
                  indent = true;
              } while(ci.Next());
          }
          printf("---------------------------------------------\n");
          delete[] symbol;
      } while((ri->Next(level)));
  }
```

# Compiling C++ API programs on Linux

Including and linking to Tesseract's API is done in a standard Linux way. To compile a basic program against the API, you can use a command like this:

```
g++ -o myprogram myprogram.cpp -llept -ltesseract
```

If Tesseract is installed in an unusual place, you can specify the include and lib directories explicitly with g++'s -I and -L flags, like this:

```
g++ -o myprogram myprogram.cpp -I/home/nick/local/include/tesseract -L/home/nick/local/lib -llept -ltesseract
```

# C-API in python

Tesseract-ocr from version 3.02.02 provide C-API. This enable to [use tesseract-ocr shared library in python](https://github.com/tesseract-ocr/tesseract/blob/master/contrib/tesseract-c_api-demo.py) (and other languages that can use C libraries):

```
import os
import ctypes

lang = "eng"
filename = "/usr/src/tesseract-ocr/phototest.tif"
libname = "/usr/local/lib64/libtesseract.so.3"
TESSDATA_PREFIX = os.environ.get('TESSDATA_PREFIX')
if not TESSDATA_PREFIX:
    TESSDATA_PREFIX = "../"

tesseract = ctypes.cdll.LoadLibrary(libname)
tesseract.TessVersion.restype = ctypes.c_char_p
tesseract_version = tesseract.TessVersion()
api = tesseract.TessBaseAPICreate()
rc = tesseract.TessBaseAPIInit3(api, TESSDATA_PREFIX, lang)
if (rc):
    tesseract.TessBaseAPIDelete(api)
    print("Could not initialize tesseract.\n")
    exit(3)

text_out = tesseract.TessBaseAPIProcessPages(api, filename, None, 0)
result_text = ctypes.string_at(text_out)

print 'Tesseract-ocr version', tesseract_version
print result_text
```

Example of passing python file object to C-API can be found at [pastebin](http://pastebin.com/yDTkNfNm).

Example of extracting orientation (from [pyocr](https://github.com/jflesch/pyocr/blob/master/src/pyocr/libtesseract/tesseract_raw.py)):
```
# ... /snip
class OSResults(Structure):
    _fields_ = [
        ('orientations', c_float * 4),
        ('scripts_na', c_float * 4 * (116 + 1 + 2 + 1)),
        ('unicharset', c_void_p),
        ('best_orientation_id', c_int),
        ('best_script_id', c_int),
        ('best_sconfidence', c_float),
        ('best_oconfidence', c_float),
        ('padding', c_char_p * 512),
    ]

# ... /snip

def create_tess_api(prefix=TESSDATA_PREFIX, lang='eng'):
    # ... /snip
    tesseract.TessBaseAPIDetectOS.argtypes = [base_api, POINTER(OSResults)]
    tesseract.TessBaseAPIDetectOS.restype = c_bool
    # ... /snip

def get_orientation(tesseract, leptonica, api, path, mode=0):
    tesseract.TessBaseAPISetPageSegMode(api, mode)
    pix = leptonica.pixRead(path)
    tesseract.TessBaseAPISetImage2(api, pix)
    osr = OSResults()
    it = tesseract.TessBaseAPIDetectOS(api, byref(osr))

    if it and osr:
        orientation, direction, line_order = c_int(), c_int(), c_int()
        skew = c_float()

        tesseract.TessPageIteratorOrientation(
            it, byref(orientation), byref(direction), byref(line_order),
            byref(skew))

        print('%s: %s' % (path, osr.best_orientation_id))
        print('confidence: %s' % osr.best_oconfidence)
```

# Example using the C-API in a C program

The C-API can of course also be used by regular C programs, as in this very basic example.

```
#include <stdio.h>
#include <allheaders.h>
#include <capi.h>

void die(const char *errstr) {
	fputs(errstr, stderr);
	exit(1);
}

int main(int argc, char *argv[]) {
	TessBaseAPI *handle;
	PIX *img;
	char *text;

	if((img = pixRead("img.png")) == NULL)
		die("Error reading image\n");

	handle = TessBaseAPICreate();
	if(TessBaseAPIInit3(handle, NULL, "eng") != 0)
		die("Error initialising tesseract\n");

	TessBaseAPISetImage2(handle, img);
	if(TessBaseAPIRecognize(handle, NULL) != 0)
		die("Error in Tesseract recognition\n");

	if((text = TessBaseAPIGetUTF8Text(handle)) == NULL)
		die("Error getting text\n");

	fputs(text, stdout);

	TessDeleteText(text);
	TessBaseAPIEnd(handle);
	TessBaseAPIDelete(handle);
	pixDestroy(&img);

	return 0;
}
```

On Linux you can [compile it as you would build a program using the C++ API](#compiling-c-api-programs-on-linux).