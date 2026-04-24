# UROVO Print Service - Android Standard Print Service

PrinterService is a print service application based on the Android standard print framework, supporting built-in printers and Bluetooth printers, providing print capabilities for web pages, applications, and other scenarios.

## Version Differentiation
1. `MORE`: Supports page-rendering paper size selection, suitable for situations where the webpage does not change and the print service adapts.
2. `A5`: Page is fixed at A5 size, webpage can be adapted, avoiding users switching page size selection.

## Features

- **Built-in Printer Support**: Support for POS built-in thermal/label printers
- **Bluetooth Printer Support**: Support for Bluetooth connected external printers
- **Cross-platform Calling**: Web pages, Android apps, React Native, Flutter, etc.
- **Standard Print Framework**: Based on Android Print Framework

## Installation and Configuration

### 1. Install Application

```bash
adb install PrinterService_v1.0.A5_XXXXXX_release.apk
```

### 2. Enable Print Service

1. Open **PrinterService** after installing the APK
2. Grant necessary permissions

### 3. Configure Printer

#### Built-in Printer (POS)

1. Open PrinterService app
2. Select device type as **POS (Built-in)**
3. Select print mode (Label/Thermal)
4. Configure paper parameters and save

#### Bluetooth Printer

1. Open PrinterService app
2. Select device type as **Bluetooth**
3. Get printer MAC address via one of the following methods:
   - Click **Scan Input** to scan printer QR code
   - Click **Search Device** to scan and select Bluetooth device
   - Manually enter MAC address (format: `AA:BB:CC:DD:EE:FF`)
4. Configure paper type and instruction type
5. Click **Connect Bluetooth** to test connection
6. Save configuration

## Usage Scenarios

### Scenario 1: Web Page Calling Print

Web pages can call the print service through standard Web Print API.

#### JavaScript Example

```javascript
// Method 1: Use window.print()
function printPage() {
    window.print();
}

// Method 2: Print specific content
function printContent(elementId) {
    const content = document.getElementById(elementId);
    const printWindow = window.open('', '', 'height=600,width=800');
    
    printWindow.document.write('<html><head><title>Print</title>');
    printWindow.document.write('</head><body>');
    printWindow.document.write(content.innerHTML);
    printWindow.document.write('</body></html>');
    
    printWindow.document.close();
    printWindow.print();
}

// Method 3: Print PDF file
function printPDF(pdfUrl) {
    const iframe = document.createElement('iframe');
    iframe.style.display = 'none';
    iframe.src = pdfUrl;
    document.body.appendChild(iframe);
    
    iframe.onload = function() {
        iframe.contentWindow.print();
    };
}
```

#### HTML Example

```html
<!DOCTYPE html>
<html>
<head>
    <title>Print Test</title>
    <style>
        @media print {
            /* Print styles */
            .no-print { display: none; }
            body { margin: 0; padding: 10px; }
        }
    </style>
</head>
<body>
    <div id="printArea">
        <h1>Order Details</h1>
        <p>Order No: 123456789</p>
        <p>Amount: ¥100.00</p>
    </div>
    
    <button class="no-print" onclick="window.print()">Print</button>
</body>
</html>
```

### Scenario 2: Android App Calling Print

#### Method 1: Using Android Print Framework

```java
import android.content.Context;
import android.print.PrintAttributes;
import android.print.PrintDocumentAdapter;
import android.print.PrintManager;
import android.webkit.WebView;
import android.webkit.WebViewClient;

public class PrintHelper {
    
    /**
     * Print web page content
     */
    public static void printWebPage(Context context, String htmlContent) {
        WebView webView = new WebView(context);
        webView.setWebViewClient(new WebViewClient() {
            @Override
            public void onPageFinished(WebView view, String url) {
                createPrintJob(context, view);
            }
        });
        
        webView.loadDataWithBaseURL(null, htmlContent, "text/html", "UTF-8", null);
    }
    
    /**
     * Create print job
     */
    private static void createPrintJob(Context context, WebView webView) {
        PrintManager printManager = (PrintManager) context.getSystemService(Context.PRINT_SERVICE);
        
        String jobName = "Document_" + System.currentTimeMillis();
        PrintDocumentAdapter printAdapter = webView.createPrintDocumentAdapter(jobName);
        
        PrintAttributes attributes = new PrintAttributes.Builder()
                .setMediaSize(PrintAttributes.MediaSize.ISO_A4)
                .setResolution(new PrintAttributes.Resolution("id", "label", 203, 203))
                .setMinMargins(PrintAttributes.Margins.NO_MARGINS)
                .build();
        
        printManager.print(jobName, printAdapter, attributes);
    }
}
```

#### Method 2: Print Custom View

```java
import android.graphics.Canvas;
import android.graphics.pdf.PdfDocument;
import android.print.PrintAttributes;
import android.print.pdf.PrintedPdfDocument;

public class CustomViewPrinter {
    
    /**
     * Print custom view
     */
    public static void printView(Context context, View view) {
        PrintManager printManager = (PrintManager) context.getSystemService(Context.PRINT_SERVICE);
        
        String jobName = "CustomView_" + System.currentTimeMillis();
        
        printManager.print(jobName, new PrintDocumentAdapter() {
            @Override
            public void onLayout(PrintAttributes oldAttributes, 
                                PrintAttributes newAttributes,
                                CancellationSignal cancellationSignal,
                                LayoutResultCallback callback,
                                Bundle extras) {
                
                if (cancellationSignal.isCanceled()) {
                    callback.onLayoutCancelled();
                    return;
                }
                
                PrintDocumentInfo info = new PrintDocumentInfo.Builder(jobName)
                        .setContentType(PrintDocumentInfo.CONTENT_TYPE_DOCUMENT)
                        .setPageCount(1)
                        .build();
                
                callback.onLayoutFinished(info, true);
            }
            
            @Override
            public void onWrite(PageRange[] pages,
                               ParcelFileDescriptor destination,
                               CancellationSignal cancellationSignal,
                               WriteResultCallback callback) {
                
                PdfDocument pdfDocument = new PdfDocument();
                PdfDocument.PageInfo pageInfo = new PdfDocument.PageInfo.Builder(
                        view.getWidth(), view.getHeight(), 1).create();
                
                PdfDocument.Page page = pdfDocument.startPage(pageInfo);
                view.draw(page.getCanvas());
                pdfDocument.finishPage(page);
                
                try {
                    pdfDocument.writeTo(new FileOutputStream(destination.getFileDescriptor()));
                    callback.onWriteFinished(new PageRange[]{PageRange.ALL_PAGES});
                } catch (IOException e) {
                    callback.onWriteFailed(e.toString());
                } finally {
                    pdfDocument.close();
                }
            }
        }, null);
    }
}
```

#### Method 3: Print Bitmap

```java
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.pdf.PdfDocument;

public class BitmapPrinter {
    
    /**
     * Print bitmap
     */
    public static void printBitmap(Context context, Bitmap bitmap) {
        PrintManager printManager = (PrintManager) context.getSystemService(Context.PRINT_SERVICE);
        
        String jobName = "Bitmap_" + System.currentTimeMillis();
        
        printManager.print(jobName, new PrintDocumentAdapter() {
            @Override
            public void onLayout(PrintAttributes oldAttributes,
                                PrintAttributes newAttributes,
                                CancellationSignal cancellationSignal,
                                LayoutResultCallback callback,
                                Bundle extras) {
                
                PrintDocumentInfo info = new PrintDocumentInfo.Builder(jobName)
                        .setContentType(PrintDocumentInfo.CONTENT_TYPE_PHOTO)
                        .setPageCount(1)
                        .build();
                
                callback.onLayoutFinished(info, true);
            }
            
            @Override
            public void onWrite(PageRange[] pages,
                               ParcelFileDescriptor destination,
                               CancellationSignal cancellationSignal,
                               WriteResultCallback callback) {
                
                PdfDocument pdfDocument = new PdfDocument();
                PdfDocument.PageInfo pageInfo = new PdfDocument.PageInfo.Builder(
                        bitmap.getWidth(), bitmap.getHeight(), 1).create();
                
                PdfDocument.Page page = pdfDocument.startPage(pageInfo);
                Canvas canvas = page.getCanvas();
                canvas.drawBitmap(bitmap, 0, 0, null);
                pdfDocument.finishPage(page);
                
                try {
                    pdfDocument.writeTo(new FileOutputStream(destination.getFileDescriptor()));
                    callback.onWriteFinished(new PageRange[]{PageRange.ALL_PAGES});
                } catch (IOException e) {
                    callback.onWriteFailed(e.toString());
                } finally {
                    pdfDocument.close();
                }
            }
        }, null);
    }
}
```

### Scenario 3: Cross-app Calling

#### Call Print via Intent

```java
import android.content.Intent;
import android.net.Uri;

public class CrossAppPrinter {
    
    /**
     * Print file (PDF, images, etc.)
     */
    public static void printFile(Context context, Uri fileUri) {
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setDataAndType(fileUri, "application/pdf");
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        
        // Trigger print dialog
        if (intent.resolveActivity(context.getPackageManager()) != null) {
            context.startActivity(intent);
        } else {
            Toast.makeText(context, "No available print service found", Toast.LENGTH_SHORT).show();
        }
    }
    
    /**
     * Share to print service
     */
    public static void shareToPrint(Context context, String text) {
        Intent shareIntent = new Intent(Intent.ACTION_SEND);
        shareIntent.setType("text/plain");
        shareIntent.putExtra(Intent.EXTRA_TEXT, text);
        
        Intent chooser = Intent.createChooser(shareIntent, "Select print method");
        context.startActivity(chooser);
    }
}
```

### Scenario 4: React Native / Flutter Integration

#### React Native Example

```javascript
// Using react-native-print library
import RNPrint from 'react-native-print';

async function printHTML() {
    const html = `
        <html>
            <body>
                <h1>Order Details</h1>
                <p>Order No: 123456789</p>
            </body>
        </html>
    `;
    
    await RNPrint.print({ html });
}

async function printPDF() {
    await RNPrint.print({
        filePath: '/path/to/document.pdf'
    });
}
```

#### Flutter Example

```dart
// Using printing package
import 'package:printing/printing.dart';
import 'package:pdf/pdf.dart';
import 'package:pdf/widgets.dart' as pw;

Future<void> printDocument() async {
  final pdf = pw.Document();
  
  pdf.addPage(
    pw.Page(
      build: (context) => pw.Center(
        child: pw.Text('Hello PrinterService!'),
      ),
    ),
  );
  
  await Printing.layoutPdf(
    onLayout: (format) => pdf.save(),
  );
}
```

## Configuration Parameters

### POS Print Mode

- **Label Mode**: Suitable for POS label paper printing
- **Thermal Mode**: Suitable for POS thermal paper printing

### Bluetooth Printer Paper Type

- **Continuous Thermal Paper**: Continuous paper without gaps
- **Label Paper**: Label paper with gaps
- **Black Mark Paper**: Paper with black marks

### Bluetooth Printer Instruction Type

- **ESC Instructions**: ESC/POS standard instruction set
- **CPCL Instructions**: Common Printer Command Language

### Bluetooth Printer Paper Size

- **Width**: Paper width (pixels)
- **Height**: Paper height (pixels)

## Error Handling

### Common Error Codes

| Error Code | Description | Solution |
|------------|-------------|----------|
| -96 | Bluetooth connection failed | Check if Bluetooth is on, verify MAC address is correct |
| -97 | Invalid MAC address | Check MAC address format (AA:BB:CC:DD:EE:FF) |
| -98 | Label printing not supported | Switch to thermal print mode |
| -99 | Data parsing error | Check print content format |
| PRNSTS_BUSY | Printer busy | Wait for current task to complete |
| PRNSTS_OUT_OF_PAPER | Out of paper | Add printing paper |
| PRNSTS_OVER_HEAT | Overheating | Wait for printer to cool down |
| PRNSTS_UNDER_VOLTAGE | Insufficient voltage | Check power connection |

## Technical Architecture

```
┌─────────────────────────────────────┐
│      Application Layer              │
│   Web, Android App, Cross-platform  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   Android Print Framework           │
│   (PrintManager, PrintService API)  │
└──────────────┬──────────────────────┘
               │
               ▼
┌────────────────────────────────────┐
│      PrinterService                │
│  ┌─────────────────────────────┐   │
│  │  Printer (PrintService)     │   │
│  │  - onPrintJobQueued()       │   │
│  │  - PDF to Bitmap            │   │
│  └──────────┬──────────────────┘   │
│             │                      │
│             ▼                      │
│  ┌─────────────────────────────┐   │
│  │  PrinterFactory             │   │
│  └──────────┬──────────────────┘   │
│             │                      │
│      ┌──────┴──────┐               │
│      ▼             ▼               │
│  ┌────────┐   ┌─────────┐          │
│  │ Inner  │   │   BT    │          │
│  │Printer │   │ Printer │          │
│  └────┬───┘   └────┬────┘          │
└───────┼────────────┼───────────────┘
        │            │
        ▼            ▼
   ┌────────┐    ┌──────────┐
   │Built-in│    │Bluetooth │
   │Printer │    │ Printer  │
   └────────┘    └──────────┘
```

## Important Notes

1. **Service Installation**: APK needs to be pre-installed or manually installed by user
2. **Permission Requirements**: Bluetooth printing requires Bluetooth permission
3. **Standard Framework**: All print calls are made through Android standard Print Framework
4. **Content Format**: Service receives print documents in PDF format
5. **Configuration Priority**: Printer must be configured in PrinterService app before normal printing

## Technical Support

For technical support, please contact UROVO technical support team.

## License

Copyright © UROVO Technology Co., Ltd.
