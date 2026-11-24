## QR Code Protocol (Smart Payload)

A unified format for generating labels on the server and parsing them via camera on Android:  
* Item: `wms://itm/{id_or_code}`
* Zone / Bin (Cell): `wms://cel/{code}`
* Container / LPN: `wms://cnt/{code}`
* *Android Behavior:* The prefix allows the system to immediately identify the object type (e.g., it will prevent an item from being scanned into a "Target Bin" field). Standard manufacturer barcodes (such as EAN-13) are sent to the API as-is.