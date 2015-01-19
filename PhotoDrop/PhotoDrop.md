PhotoDrop
=========

One of the cool features of Couchbase Lite that hasn’t been featured much is an ability to do P2P replication between two devices. Couchbase Lite is packaged with an extra component called `Couchbase Lite Listener` that allows your application to accept HTTP connections from other devices running Couchbase Lite and exchange data with them in either push or pull direction.

A couple months ago, Wayne Carter and Traun Leyden skyped me in and sold me an idea of creating a simple P2P photo sharing app that can demonstrate how easily we can develop a P2P application by using Couchbase Lite. I was engaged  since then and spent a couple days working on this little P2P iOS photo sharing app, and I called it `PhotoDrop`. In this blog post, I will show you how I use Couchbase Lite to develop the application.

Overview
--------
`PhotoDrop` is a P2P photo sharing app similar to the iOS `AirDrop` feature that you can send photos across different devices. P2P discovery is a difficult subject especially when combining with identity and security issues. To make things simple, I use the classic QRCode for discovering other peers and ignore both identity and security matter all together. The flow of the application is fairly simple. You select photos you want to share to your friend and open the QRCode scanner to scan the target endpoint that the photo will be sent to. On another side, your friend opens the appliation, shows up the QRCode and waits for you to scan and send the photos.

Selecting photos
----------------
`PhotoDrop` uses ALAssetsLibrary to access to the photos in the Camera Roll album on an iOS device instead of using the UIImagePickerViewController that doesn't support multiple photo selection.

```Swift
	// ViewController.swift

	// MARK: - ALAssetsLibrary
    func reloadAssets() {
        if library == nil {
            library = ALAssetsLibrary()
        }

        assets.removeAll(keepCapacity: true)
        library.enumerateGroupsWithTypes(ALAssetsGroupSavedPhotos, usingBlock:
            { (group:ALAssetsGroup!, stop:UnsafeMutablePointer<ObjCBool>) -> Void in
                if group != nil {
                    group.setAssetsFilter(ALAssetsFilter.allPhotos())
                    group.enumerateAssetsWithOptions(NSEnumerationOptions.Reverse,
                        usingBlock: { (asset:ALAsset!, index:Int,
                            stop: UnsafeMutablePointer<ObjCBool>) -> Void in
                            if asset != nil {
                                self.assets.append(asset)
                            }
                    })
                } else {
                    dispatch_async(dispatch_get_main_queue(), {
                        self.collectionView.reloadData()
                    })
                }
            }) { (error:NSError!) -> Void in
        }
    }
```
The photo assets are displayed in a UICollectionViewController. A simple custom UICollectionViewCell named `PhotoViewCell` is created to display a photo thumbnail and a checkbox indicating the selecting status of the photo.

```Swift
	// ViewController.swift

	// MARK: - UICollectionView
    func collectionView(collectionView: UICollectionView,
        cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCellWithReuseIdentifier("photoCell",
            forIndexPath: indexPath) as PhotoViewCell
        let asset = assets[indexPath.row]
        cell.imageView.image = UIImage(CGImage: asset.thumbnail().takeUnretainedValue())
        cell.checked = contains(selectedAssets, asset)
        return cell
    }

    func collectionView(collectionView: UICollectionView,
        didSelectItemAtIndexPath indexPath: NSIndexPath) {
        let cell = collectionView.cellForItemAtIndexPath(indexPath) as PhotoViewCell
        let asset = assets[indexPath.row]
        if let foundIndex = find(selectedAssets, asset) {
            selectedAssets.removeAtIndex(foundIndex)
            cell.checked = false
        } else {
            selectedAssets.append(asset)
            cell.checked = true
        }
        collectionView.reloadItemsAtIndexPaths([indexPath])

        self.enableSendMode(selectedAssets.count > 0)
    }
```

Sending photos
--------------
When the selected photos are ready to send as the send button is touched, the SendViewController will be  presented and set with the selected photos.

```Swift
	// ViewController.swift

    override func prepareForSegue(segue: UIStoryboardSegue, sender: AnyObject?) {
        if segue.identifier == "send" {
            let navigationController = segue.destinationViewController as UINavigationController
            let controller = navigationController.topViewController as SendViewController
            controller.sharedAssets = selectedAssets
        }
    }
```
Upon the SendViewController is presented, we find ourselves in the viewDidLoad() function in which we get an empty database object named "db" from the `DatabaseUtil` class. The reason that we get a freash database as to ensure that there are no pending documents from a previous sharing session.

```Swift
	// SendViewController.swift

    override func viewDidLoad() {
        super.viewDidLoad()

        var error: NSError?
        database = DatabaseUtil.getEmptyDatabase("db", error: &error)
        if error != nil {
            AppDelegate.showMessage("Cannot get a database with error : \(error!.code)", title: "Error")
        }
    }
```
The getEmptyDatabase() function of the DatabaseUtil class is ensure to return an empty database with the given name by deleting the database if it does exist and recreating a new one.

```Swift
	// DatabaseUtil.swift

	class func getEmptyDatabase(name: String!, error: NSErrorPointer) -> CBLDatabase? {
        if let database = CBLManager.sharedInstance().existingDatabaseNamed(name, error: nil) {
            if !database.deleteDatabase(error) {
                return nil;
            }
        }
        return CBLManager.sharedInstance().databaseNamed(name, error: error)
    }
```
Once the SendViewController is presented, we will be at the viewDidAppear(animated:Bool) function. At the viewDidAppear function, we start the AVFoundation's QRCode capturing session to present the QRCode scanner.

```Swift
	// SendViewController.swift

	override func viewDidAppear(animated: Bool) {
        super.viewDidAppear(animated)

        if database != nil && session == nil {
            startCaptureSession()
        }
    }

	func startCaptureSession() {
        let app = UIApplication.sharedApplication().delegate as AppDelegate

        let device = AVCaptureDevice.defaultDeviceWithMediaType(AVMediaTypeVideo)
        if device == nil {
            AppDelegate.showMessage("No video capture devices found", title: "")
            return
        }

        var error: NSError?
        let input = AVCaptureDeviceInput.deviceInputWithDevice(device, error: &error)
            as AVCaptureDeviceInput
        if error == nil {
            session = AVCaptureSession()
            session.addInput(input)
            let output = AVCaptureMetadataOutput()
            output.setMetadataObjectsDelegate(self, queue: dispatch_get_main_queue())
            session.addOutput(output)
            output.metadataObjectTypes = [AVMetadataObjectTypeQRCode]

            previewLayer = AVCaptureVideoPreviewLayer.layerWithSession(session)
                as AVCaptureVideoPreviewLayer
            previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill
            previewLayer.frame = self.previewView.bounds
            self.previewView.layer.addSublayer(previewLayer)

            session.startRunning()
        } else {
            AppDelegate.showMessage("Cannot start QRCode capture session", title: "Error")
        }
    }
```
When a QRCode is captured, we extract the endpoint URL which is the remote URL that we will send the photo documents to. The core code for creating and sending photo documents begins and ends just right here in the replicate(url: NSURL) function which has about 50 lines of code.

The code starts with looping thru each photo in the sharedAssets array. For each photo, we get a binary representation and attach to an empty `Couchbase Lite` document created correspondingly.

```Swift
	// SendViewController.swift

	func replicate(url: NSURL) {
        self.previewView.hidden = true;
        self.statusLabel.text = "Sending Photos ..."
        UIApplication.sharedApplication().networkActivityIndicatorVisible = true

        var docIds: [String] = []
        for asset in sharedAssets! {
            let representation = asset.defaultRepresentation()
            var bufferSize = UInt(Int(representation.size()))
            var buffer = UnsafeMutablePointer<UInt8>(malloc(bufferSize))
            var buffered = representation.getBytes(buffer, fromOffset: 0,
                length: Int(representation.size()), error: nil)
            var data = NSData(bytesNoCopy: buffer, length: buffered, freeWhenDone: true)

            var error: NSError?
            let doc = database.createDocument()
            let rev = doc.newRevision()
            rev.setAttachmentNamed("photo", withContentType: "application/octet-stream", content: data)
            let saved = rev.save(&error)

            if saved != nil {
                docIds.append(doc.documentID)
            }
        }
		...
    }
```
After we finish creating the documents as above code snippet, we are now going to send those documents with photo attached to the other device. We create a push replicator and set the ids of the documents that we would like to replicate over to the other device. In general, setting the document ids to a replicator is optional in which the replicator will just replicate all of the documents in the database.

Before we start the replicator, we setup a notification observer to observe the replicating status so that we can display replication status appropriately when the replication is started and done.

```Swift
	// SendViewController.swift

	func replicate(url: NSURL) {
    	...
		if docIds.count > 0 {
            replicator = database.createPushReplication(url)
            replicator.documentIDs = docIds

            NSNotificationCenter.defaultCenter().addObserverForName(kCBLReplicationChangeNotification,
                object: replicator, queue: nil) { (notification) -> Void in
                    if self.replicator.lastError == nil {
                        var totalCount = self.replicator.changesCount
                        var completedCount = self.replicator.completedChangesCount
                        if completedCount > 0 && completedCount == totalCount {
                            self.statusLabel.text = "Sending Completed"
                            UIApplication.sharedApplication().networkActivityIndicatorVisible = false
                        }
                    } else {
                        self.statusLabel.text = "Sending Abort"
                        UIApplication.sharedApplication().networkActivityIndicatorVisible = false
                    }
            }
            replicator.start()
        }
	}
```

Receiving photos
----------------

Receiving photos is done in `ReceiveViewController` which is presented from the ViewController screen when a user touches the Receive button. When the ReceiveViewController is presented, at viewDidLoad() method, similarly to the `SendViewController`, we get a fresh database named "db". At the viewDidAppear(animated: Bool) function, we call startListener() to create and start a `CBLListener` object. The `CBLListener` is an embedded lightweight HTTP server which listens to HTTP requests and routes those requests to appropriate handlers to perform replicating operations. Once the listener is successfully started, we can obtain the listener's URL and present a QRCode encoding the URL. We are lucky that the iOS CoreImage supports a QRCode filter so that we could generate a QRCode image in very less effort.

```Swift
	// ReceiveViewController.swift

	override func viewDidLoad() {
        super.viewDidLoad()

        collectionView.hidden = true;

        var error: NSError?
        database = DatabaseUtil.getEmptyDatabase("db", error: &error)
        if error != nil {
            AppDelegate.showMessage("Cannot get a database with error : \(error!.code)", title: "Error")
        }
    }

    override func viewDidAppear(animated: Bool) {
        super.viewDidAppear(animated)

        if database == nil {
            return;
        }

        startListener()

        if let url = syncUrl()?.absoluteString {
            imageView.image = UIImage.qrCodeImageForString(url,
                size: imageView.frame.size)
        }
    }

	// MARK: - Listener
    func startListener() {
        if listener != nil {
            return
        }

        var error: NSError?
        listener = CBLListener(manager: CBLManager.sharedInstance(), port: 0)
        var success = listener.start(&error)
        if success {
            startObserveDatabaseChange()
        } else {
            listener = nil
            AppDelegate.showMessage("Cannot start listener", title: "Error")
        }
    }
```

As you can see above, after starting the listener, the startObserveDatabaseChange() function is called to observe any changes happening to the database via notifications named `kCBLDatabaseChangeNotification`. We are now able to know when the photo documents from the other device are replicated so that we could save the photos into the device's camera roll accordingly.

```Swift
	// ReceiveViewController.swift

	// MARK: - Database Change
    func startObserveDatabaseChange() {
        NSNotificationCenter.defaultCenter().addObserverForName(kCBLDatabaseChangeNotification,
            object: database, queue: nil) {
                (notification) -> Void in
                if let changes = notification.userInfo!["changes"] as? [CBLDatabaseChange] {
                    for change in changes {
                        dispatch_async(dispatch_get_main_queue(), {
                            if self.collectionView.hidden {
                                self.collectionView.hidden = false
                            }
                            self.saveImageFromDocument(change.documentID)
                        })
                    }
                }
        }
    }
```

The function to save a photo into to the device's camera roll is as below. The function simply gets the image from the attachment of the document, creates an CGImage representation and saves to the camera roll album via the ALAssetLibrary's writeImageDataToSavedPhotosAlbum() function. After saving a photo, we display the thumbnail version of the photo on the screen.

```Swift
	// ReceiveViewController.swift

	func saveImageFromDocument(docId: String) {
        let app = UIApplication.sharedApplication().delegate as AppDelegate
        if let doc = app.database.existingDocumentWithID(docId) {
            if doc.currentRevision.attachments.count > 0 {
                let attachment = doc.currentRevision.attachments[0] as CBLAttachment
                if let image = UIImage(data: attachment.content)?.CGImage {
                    let library = assetsLibrary()
                    library.writeImageDataToSavedPhotosAlbum(attachment.content, metadata: nil,
                        completionBlock: { (url: NSURL!, error: NSError!) -> Void in
                        if url != nil {
                            library.assetForURL(url, resultBlock:
                                {(asset: ALAsset!) -> Void in
                                    self.assets.insert(asset, atIndex: 0)
                                    dispatch_async(dispatch_get_main_queue(), {
                                        self.collectionView.insertItemsAtIndexPaths(
                                            [NSIndexPath(forRow: 0, inSection: 0)])
                                    })
                                })
                                {(error: NSError!) -> Void in
                            }
                        }
                    })
                }
            }
        }
    }
```


Wrapping up
-----------

There are many ways to develop `PhotoDrop` app, and using `Couchbase Lite` is one way and perhaps one of the easiest ways to do it. Honestly the core code for sending and receiving photos is less than 100 lines of code with zero lines of code directly involving network or HTTP communication.

Where to go from here? Recently Jens Alfke added the SSL support including a self-signed certificate generating API to the `Couchbase Lite Listener` so we could easily secure the communication channels during the replication. We could look more into the Bonjour service discovery API, which is actually already supported by the listener. Expanding to other platforms such as Android or even OS X could also be interesting.

I hope this post give you some ideas about how to use Couchbase Lite to do P2P replication. Clone the [PhotoDrop](https://github.com/couchbaselabs/PhotoDrop-iOS) GitHub repo and play with it yourself. Let us know what you think.