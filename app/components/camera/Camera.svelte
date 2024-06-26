<script lang="ts">
    import { CameraView } from '@nativescript-community/ui-cameraview';
    import { Canvas, CanvasView, Paint, Style } from '@nativescript-community/ui-canvas';
    import { Img } from '@nativescript-community/ui-image';
    import { showBottomSheet } from '@nativescript-community/ui-material-bottomsheet/svelte';
    import { confirm } from '@nativescript-community/ui-material-dialogs';
    import { showSnack } from '@nativescript-community/ui-material-snackbar';
    import { AndroidActivityBackPressedEventData, Application, ApplicationSettings, CoreTypes, Page, TouchAnimationOptions, Utils, knownFolders, path } from '@nativescript/core';
    import { ImageSource } from '@nativescript/core/image-source';
    import dayjs from 'dayjs';
    import {
        CropResult,
        createAutoScanHandler,
        cropDocument,
        cropDocumentFromFile,
        detectQRCode,
        getColorPalette,
        getJSONDocumentCorners,
        getJSONDocumentCornersFromFile,
        processFromFile
    } from 'plugin-nativeprocessor';
    import { CropView } from 'plugin-nativeprocessor/CropView';
    import { onDestroy, onMount } from 'svelte';
    import { closeModal, showModal } from 'svelte-native';
    import { NativeViewElementNode } from 'svelte-native/dom';
    import { navigate } from '~/utils/svelte/ui';
    import { writable } from 'svelte/store';
    import CameraSettingsBottomSheet from '~/components/camera/CameraSettingsBottomSheet.svelte';
    import CActionBar from '~/components/common/CActionBar.svelte';
    import { l, lc } from '~/helpers/locale';
    import { OCRDocument, PageData } from '~/models/OCRDocument';
    import {
        AUTO_SCAN_DELAY,
        AUTO_SCAN_DISTANCETHRESHOLD,
        AUTO_SCAN_DURATION,
        AUTO_SCAN_ENABLED,
        CROP_ENABLED,
        DOCUMENT_NOT_DETECTED_MARGIN,
        IMAGE_CONTEXT_OPTIONS,
        IMG_COMPRESS,
        IMG_FORMAT,
        PREVIEW_RESIZE_THRESHOLD,
        QRCODE_RESIZE_THRESHOLD,
        TRANSFORMS_SPLIT
    } from '~/models/constants';
    import { documentsService } from '~/services/documents';
    import { showError, wrapNativeException } from '~/utils/error';
    import { getColorMatrix, hideLoading, onBackButton, showLoading, showSettings } from '~/utils/ui';
    import { recycleImages } from '~/utils/images';
    import { colors, navigationBarHeight } from '~/variables';
    import IconButton from '~/components/common/IconButton.svelte';
    import { debounce } from '@nativescript/core/utils';

    // technique for only specific properties to get updated on store change
    $: ({ colorPrimary } = $colors);

    const touchAnimationShrink: TouchAnimationOptions = {
        down: {
            scale: { x: 0.9, y: 0.9 },
            // backgroundColor: colorPrimary.darken(10),
            duration: 100,
            curve: CoreTypes.AnimationCurve.easeInOut
        },
        up: {
            scale: { x: 1, y: 1 },
            // backgroundColor: colorPrimary,
            duration: 100,
            curve: CoreTypes.AnimationCurve.easeInOut
        }
    };
    let page: NativeViewElementNode<Page>;
    let cameraView: NativeViewElementNode<CameraView>;
    let takPictureBtnCanvas: NativeViewElementNode<CanvasView>;
    let cropView: NativeViewElementNode<CropView>;
    let fullImageView: NativeViewElementNode<Img>;
    let smallImageView: NativeViewElementNode<Img>;

    const cameraOptionsStore = writable<{ aspectRatio: string; stretch: string; viewsize: string; pictureSize: string }>(
        JSON.parse(ApplicationSettings.getString('camera_settings', '{"aspectRatio":"4:3", "stretch":"aspectFit","viewsize":"limited", "pictureSize":null}'))
    );
    cameraOptionsStore.subscribe((newValue) => {
        ApplicationSettings.setString('camera_settings', JSON.stringify(newValue));
    });
    $: ({ aspectRatio, stretch, viewsize, pictureSize } = $cameraOptionsStore);
    $: DEV_LOG && console.log('aspectRatio', aspectRatio);
    $: DEV_LOG && console.log('pictureSize', pictureSize);

    // let aspectRatio = ApplicationSettings.getString('camera_aspectratio', '4:3');
    // let stretch = ApplicationSettings.getString('camera_stretch', 'aspectFit');
    // let collectionView: NativeViewElementNode<CollectionView>;

    export let modal = false;
    export let document: OCRDocument = null;
    // export let outputUri; // android only
    // export let forActivityResult = false; //android only

    const contours: [number, number][][] = null;
    // let pages: ObservableArray<OCRPage>;
    let nbPages = 0;
    let takingPicture = false;
    // let croppedImage: string | ImageSource = null;
    let smallImage: string = null;
    let smallImageRotation: number = 0;
    // let croppedImageRotation: number = 0;
    const noDetectionMargin = ApplicationSettings.getNumber('documentNotDetectedMargin', DOCUMENT_NOT_DETECTED_MARGIN);
    const previewResizeThreshold = ApplicationSettings.getNumber('previewResizeThreshold', PREVIEW_RESIZE_THRESHOLD);
    let colorType = ApplicationSettings.getString('defaultColorType', 'normal');
    let colorMatrix = JSON.parse(ApplicationSettings.getString('defaultColorMatrix', null));
    let transforms = ApplicationSettings.getString('defaultTransforms', '').split(TRANSFORMS_SPLIT);
    let flashMode = ApplicationSettings.getNumber('defaultFlashMode', 0);
    const zoom = ApplicationSettings.getNumber('defaultZoom', 1);
    let _actualFlashMode = flashMode;
    let torchEnabled = false;
    let batchMode = ApplicationSettings.getBoolean('batchMode', false);
    let canSaveDoc = false;
    let cameraScreenRatio = 1;
    const showingFullScreenImage = false;
    let editing = false;

    const startOnCam = ApplicationSettings.getBoolean('startOnCam', START_ON_CAM) && !modal;
    $: ApplicationSettings.setBoolean('batchMode', batchMode);

    // interface Option {
    //     icon: string;
    //     id: string;
    //     text: string;
    // }
    async function showDocumentsList() {
        if (CARD_APP) {
            const CardsList = (await import('~/components/CardsList.svelte')).default;
            return navigate({ page: CardsList });
        } else {
            const DocumentsList = (await import('~/components/DocumentsList.svelte')).default;
            return navigate({ page: DocumentsList });
        }
    }

    async function goToView(doc: OCRDocument) {
        if (CARD_APP) {
            const page = (await import('~/components/view/CardView.svelte')).default;
            return navigate({
                page,
                props: {
                    document: doc
                }
            });
        } else {
            const page = (await import('~/components/view/DocumentView.svelte')).default;
            return navigate({
                page,
                props: {
                    document: doc
                }
            });
        }
    }
    // async function showOptions() {
    //     const result: { icon: string; id: string; text: string } = await showBottomSheet({
    //         parent: page,
    //         view: ActionSheet,
    //         props: {
    //             options: [
    //                 {
    //                     icon: 'mdi-cogs',
    //                     id: 'preferences',
    //                     text: l('preferences')
    //                 },
    //                 {
    //                     icon: 'mdi-image-plus',
    //                     id: 'image_import',
    //                     text: l('image_import')
    //                 }
    //             ]
    //         }
    //     });
    //     if (result) {
    //         switch (result.id) {
    //             case 'preferences':
    //                 prefs.openSettings();
    //                 break;
    //             case 'image_import':
    //                 try {
    //                     const doc = await importAndScanImage();
    //                     await goToView(doc);
    //                 } catch (err) {
    //                     console.error(err);
    //                 }

    //                 break;
    //         }
    //     }
    // }

    async function showCameraSettings() {
        const addedProps: any = __ANDROID__
            ? {
                  resolutions: cameraView.nativeView.getAllAvailablePictureSizes(),
                  currentResolution: cameraView.nativeView.getCurrentResolutionInfo()
              }
            : {};
        const result: { icon: string; id: string; text: string } = await showBottomSheet({
            parent: page,
            view: CameraSettingsBottomSheet,
            backgroundOpacity: 0.8,
            skipCollapsedState: true,
            closeCallback: (result, bottomsheetComponent: CameraSettingsBottomSheet) => {
                transforms = bottomsheetComponent.transforms;
                colorType = bottomsheetComponent.colorType;
                colorMatrix = bottomsheetComponent.colorMatrix;
                ApplicationSettings.setString('defaultColorType', colorType);
                if (colorMatrix) {
                    ApplicationSettings.setString('defaultColorMatrix', JSON.stringify(colorMatrix));
                } else {
                    ApplicationSettings.remove('defaultColorMatrix');
                }
                ApplicationSettings.setString('defaultTransforms', transforms.join(TRANSFORMS_SPLIT));
            },
            props: {
                cameraOptionsStore,
                colorType,
                colorMatrix,
                transforms,
                ...addedProps
            }
        });
    }

    function onImageTap(e, item) {
        setCurrentImage(e.object.src);
    }

    // let editingImage: ImageSource;

    async function processAndAddImage(image, autoScan = false) {
        try {
            showLoading(l('computing'));
            let imageSource = new ImageSource(image);

            const imageWidth = imageSource.width;
            const imageHeight = imageSource.height;
            const imageRotation = imageSource.rotationAngle;
            const cropEnabled = ApplicationSettings.getBoolean('cropEnabled', CROP_ENABLED);
            let quads: [number, number][][];

            const tempImagePath = path.join(knownFolders.temp().path, `capture_${Date.now()}.jpg`);
            await imageSource.saveToFileAsync(tempImagePath, IMG_FORMAT, IMG_COMPRESS);
            recycleImages(imageSource);
            imageSource = null;
            // TODO: we need to save the image to do anything
            if (cropEnabled) {
                quads = await getJSONDocumentCornersFromFile(tempImagePath, previewResizeThreshold * 1.5, 0);
            }
            DEV_LOG && console.log('processAndAddImage', tempImagePath, previewResizeThreshold, quads, autoScan, imageWidth, imageHeight);
            if (cropEnabled && quads.length === 0) {
                let items = [
                    {
                        imagePath: tempImagePath,
                        imageWidth,
                        imageHeight,
                        imageRotation,
                        quads: [
                            [
                                [noDetectionMargin, noDetectionMargin],
                                [imageWidth - noDetectionMargin, noDetectionMargin],
                                [imageWidth - noDetectionMargin, imageHeight - noDetectionMargin],
                                [noDetectionMargin, imageHeight - noDetectionMargin]
                            ]
                        ] as [number, number][][]
                    }
                ];
                if (autoScan === false) {
                    if (torchEnabled) {
                        forceTorchDisabled(true);
                    }
                    const ModalImportImage = (await import('~/components/ModalImportImages.svelte')).default;
                    items = await showModal({
                        page: ModalImportImage,
                        animated: true,
                        fullscreen: true,
                        props: {
                            items
                        }
                    });
                    quads = items ? items[0].quads : undefined;
                    if (torchEnabled) {
                        forceTorchDisabled(false);
                    }
                }
            }
            if (!cropEnabled || quads?.length) {
                await addCurrentImageToDocument(tempImagePath, imageWidth, imageHeight, imageRotation, quads);
                return true;
            }
            recycleImages(imageSource);
            showSnack({ message: lc('no_document_found') });
            return false;
        } catch (err) {
            console.error(err, err.stack);
            showError(err);
        } finally {
            takingPicture = false;
            hideLoading();
        }
    }

    function pauseAutoScan() {
        DEV_LOG && console.log('pauseAutoScan', autoScanHandler, autoScanHandler?.enabled);
        if (autoScanHandler) {
            autoScanHandler.enabled = false;
        }
    }
    function resumeAutoScan() {
        DEV_LOG && console.log('resumeAutoScan', autoScanHandler, autoScanHandler?.enabled);
        if (autoScanHandler) {
            autoScanHandler.enabled = true;
        }
    }
    async function takePicture(autoScan = false) {
        if (takingPicture) {
            return;
        }
        takingPicture = true;
        if (autoScanHandler) {
            pauseAutoScan();
        }
        try {
            DEV_LOG && console.log('takePicture', autoScan, _actualFlashMode);
            await showLoading(l('capturing'));
            const { image, info } = await cameraView.nativeView.takePicture({
                savePhotoToDisk: false,
                flashMode: _actualFlashMode,
                maxWidth: 4500,
                maxHeight: 4500
            });
            const didAdd = await processAndAddImage(image, autoScan);
            DEV_LOG && console.log('takePicture done', image, didAdd);
            if (didAdd && !batchMode) {
                await saveCurrentDocument();
            }
        } catch (err) {
            // we can get a native error here
            showError(wrapNativeException(err));
        } finally {
            takingPicture = false;
            if (autoScanHandler) {
                resumeAutoScan();
            }
            hideLoading();
        }
    }

    $: {
        _actualFlashMode = torchEnabled ? 4 : (flashMode as any);
    }
    function forceTorchDisabled(value) {
        if (value) {
            _actualFlashMode = flashMode;
        } else {
            _actualFlashMode = torchEnabled ? 4 : (flashMode as any);
        }
    }
    function switchTorch() {
        torchEnabled = !torchEnabled;
    }
    function toggleCamera() {
        cameraView.nativeView.toggleCamera();
    }

    onMount(async () => {
        onNavigatedTo();
        if (__ANDROID__) {
            Application.android.on(Application.android.activityBackPressedEvent, onAndroidBackButton);
        }
        Application.on(Application.backgroundEvent, onBackground);
        Application.on(Application.foregroundEvent, onForeground);

        if (documentsService.started) {
            startPreview();
        } else {
            documentsService.once('started', startPreview);
        }
    });
    onDestroy(() => {
        // clearImages();
        document = null;
        nbPages = 0;
        if (autoScanHandler) {
            autoScanHandler.enabled = false;
            autoScanHandler = null;
        }
        if (processor) {
            processor.autoScanHandler = null;
            processor = null;
        }
        if (__ANDROID__) {
            Application.android.off(Application.android.activityBackPressedEvent, onAndroidBackButton);
        }
        Application.off(Application.backgroundEvent, onBackground);
        Application.off(Application.foregroundEvent, onForeground);
    });
    function onNavigatedFrom() {
        if (torchEnabled) {
            forceTorchDisabled(true);
        }
        stopPreview();
        if (document) {
            // we need to clear the current document which was not saved
            //especially memory images
            document.removeFromDisk();
            document = null;
        }
    }
    let previewStarted = true;
    function startPreview() {
        if (!previewStarted) {
            previewStarted = true;
            cameraView?.nativeView.startPreview();
        }
    }
    function stopPreview() {
        if (previewStarted) {
            previewStarted = false;
            cameraView?.nativeView.stopPreview();
            if (cropView?.nativeView) {
                cropView.nativeView.quads = null;
            }
        }
    }
    function onNavigatedTo() {
        (async () => {
            try {
                startPreview();
            } catch (error) {
                console.error(error, error.stack);
            }
        })();
    }
    function onBackground() {
        stopPreview();
    }
    function onForeground() {
        DEV_LOG && console.log('onForeground', !!cameraView);
        startPreview();
    }
    async function saveCurrentDocument() {
        const newDocument = !document;
        try {
            DEV_LOG && console.log('saveCurrentDocument', newDocument, !!document);
            if (!document) {
                document = await OCRDocument.createDocument(dayjs().format('L LTS'), pagesToAdd);
                if (startOnCam) {
                    await goToView(document);
                } else {
                    // we should already be in edit so closing should go back there
                }
            } else {
                await document.addPages(pagesToAdd);
                await document.save({}, false);
            }
            if (document) {
                if (!startOnCam) {
                    closeModal(document);
                }
            }
        } catch (error) {
            showError(error);
        }
    }

    const onAndroidBackButton = (data: AndroidActivityBackPressedEventData) =>
        onBackButton(page?.nativeView, () => {
            if (editing) {
                toggleEditing();
                data.cancel = true;
            }
        });

    // function clearImages() {
    //     // if (editingImage) {
    //     const toRelease = [editingImage, smallImage].concat(pagesToAdd ? pagesToAdd.map((d) => d.image) : []);
    //     editingImage = null;
    //     smallImage = null;
    //     recycleImages(toRelease);
    // }
    const pagesToAdd: PageData[] = [];

    async function addCurrentImageToDocument(sourceImagePath, imageWidth, imageHeight, imageRotation, quads) {
        try {
            if (!sourceImagePath) {
                return;
            }
            const strTransforms = transforms.join(TRANSFORMS_SPLIT);
            DEV_LOG && console.log('addCurrentImageToDocument', sourceImagePath, quads, processor);
            const images: CropResult[] = [];
            if (quads) {
                images.push(
                    ...(await cropDocumentFromFile(sourceImagePath, quads, {
                        transforms: strTransforms,
                        saveInFolder: knownFolders.temp().path,
                        compressFormat: IMG_FORMAT,
                        compressQuality: IMG_COMPRESS
                    }))
                );
                // we generate
            } else {
                images.push({ imagePath: sourceImagePath, width: imageWidth, height: imageHeight });
            }
            // let images = quads ? await cropDocumentFromFile(sourceImagePath, quads, strTransforms) : [sourceImagePath];
            if (images.length) {
                // if (!document) {
                //     document = await OCRDocument.createDocument(dayjs().format('L LTS'));
                // }
                let qrcode;
                let colors;
                if (CARD_APP) {
                    [qrcode, colors] = await processFromFile(
                        sourceImagePath,
                        [
                            {
                                type: 'qrcode'
                            },
                            {
                                type: 'palette'
                            }
                        ],
                        {
                            maxSize: QRCODE_RESIZE_THRESHOLD
                        }
                    );
                    // Promise.all([detectQRCode(images[0], { resizeThreshold: QRCODE_RESIZE_THRESHOLD }), getColorPalette(images[0])]);
                    DEV_LOG && console.log('qrcode and colors', qrcode, colors);
                }
                for (let index = 0; index < images.length; index++) {
                    const image = images[index];
                    pagesToAdd.push({
                        ...image,
                        crop: quads?.[index] || [
                            [0, 0],
                            [imageWidth - 0, 0],
                            [imageWidth - 0, imageHeight - 0],
                            [0, imageHeight - 0]
                        ],
                        colorType,
                        colorMatrix,
                        colors,
                        qrcode,
                        transforms: strTransforms,
                        sourceImagePath,
                        sourceImageWidth: imageWidth,
                        sourceImageHeight: imageHeight,
                        sourceImageRotation: imageRotation,
                        rotation: imageRotation
                    });
                }
            }
            nbPages = pagesToAdd.length;
            startPreview();
            const lastPage = pagesToAdd[pagesToAdd.length - 1];
            setCurrentImage(lastPage.imagePath, lastPage.rotation, true);
        } catch (error) {
            showError(error);
        }
    }

    async function setCurrentImage(image: string, rotation = 0, needAnimateBack = false) {
        // const imageView = fullImageView.nativeElement;
        // const sImageView = smallImageView.nativeElement;
        // imageView.originX = 0.5;
        // imageView.originY = 0.5;
        // if (!image) {
        //     showingFullScreenImage = false;
        //     await imageView.animate({
        //         duration: 200,
        //         opacity: 0,
        //         scale: {
        //             x: 0.5,
        //             y: 0.5
        //         }
        //     });
        //     // croppedImage = image;
        //     // croppedImageRotation = rotation;
        // } else if (image) {
        smallImage = image;
        smallImageRotation = rotation;
        // smallImageView.nativeElement.opacity = 0;
        // showingFullScreenImage = true;
        // // croppedImage = image;
        // // croppedImageRotation = rotation;
        // imageView.translateX = 0;
        // imageView.translateY = 0;
        // imageView.opacity = 0;
        // imageView.scaleX = 0.5;
        // imageView.scaleY = 0.5;
        // console.log('animating', needAnimateBack);
        // await imageView.animate({
        //     duration: 200,
        //     opacity: 1,
        //     scale: {
        //         x: 1,
        //         y: 1
        //     }
        // });
        // if (needAnimateBack) {
        //     imageView.originX = 0;
        //     imageView.originY = 1;
        //     const position = sImageView.getLocationOnScreen();
        //     const size = { width: sImageView.getMeasuredWidth(), height: sImageView.getMeasuredHeight() };
        //     const ratio = imageView.getMeasuredWidth() / imageView.getMeasuredHeight();
        //     const scaleX = (ratio * size.height) / imageView.getMeasuredWidth();
        //     const scaleY = size.height / imageView.getMeasuredHeight();
        //     console.log('animateBack', showingFullScreenImage, ratio, scaleX, scaleY);
        //     await imageView.animate({
        //         duration: 400,
        //         scale: {
        //             x: scaleX,
        //             y: scaleY
        //         },

        //         translate: {
        //             x: position.x + Utils.layout.toDeviceIndependentPixels((size.width - ratio * size.height) / 2),
        //             y: -(Screen.mainScreen.heightDIPs - position.y - Utils.layout.toDeviceIndependentPixels(size.height))
        //         }
        //     });
        //     imageView.opacity = 0;
        // }
        // smallImageView.nativeElement.opacity = 1;
        // showingFullScreenImage = false;
        // }
    }
    $: canSaveDoc = nbPages > 0;

    function onCameraLayoutChanged() {
        cameraScreenRatio = cameraView.nativeElement.getMeasuredWidth() / cameraView.nativeElement.getMeasuredHeight();
    }
    function focusCamera(e) {
        DEV_LOG && console.log('focusCamera', e.getX(), e.getY());
        cameraView.nativeElement.focusAtPoint(e.getX(), e.getY());
    }
    const onZoom = debounce(function onZoom(event) {
        ApplicationSettings.setNumber('defaultZoom', event.zoom);
    }, 500);

    let autoScan = ApplicationSettings.getBoolean('autoScan', AUTO_SCAN_ENABLED);
    let processor;
    let autoScanHandler;
    function applyAutoScan(value: boolean) {
        if (value) {
            const nCropView = cropView.nativeView;
            const newAutoScanHandler = createAutoScanHandler(nCropView, (result) => {
                DEV_LOG && console.log('onAutoScan', result);
                // TODO: safeguard though should never happen
                if (cameraView?.nativeView) {
                    takePicture(true);
                }
            });
            newAutoScanHandler.distanceThreshod = ApplicationSettings.getNumber('autoScan_distanceThreshold', AUTO_SCAN_DISTANCETHRESHOLD);
            newAutoScanHandler.autoScanDuration = ApplicationSettings.getNumber('autoScan_autoScanDuration', AUTO_SCAN_DURATION);
            newAutoScanHandler.preAutoScanDelay = ApplicationSettings.getNumber('autoScan_preAutoScanDelay', AUTO_SCAN_DELAY);
            autoScanHandler = processor.autoScanHandler = newAutoScanHandler;
        } else {
            processor.autoScanHandler = null;
            autoScanHandler = null;
        }
    }
    function toggleAutoScan(apply = true) {
        DEV_LOG && console.log('toggleAutoScan', autoScan, apply);
        autoScan = !autoScan;
        ApplicationSettings.setBoolean('autoScan', autoScan);
        if (apply) {
            applyAutoScan(autoScan);
        }
    }
    async function applyProcessor() {
        if (processor) {
            return;
        }

        const showAutoScanWarning = ApplicationSettings.getBoolean('showAutoScanWarning', true);
        if (showAutoScanWarning) {
            if (autoScan) {
                const result = await confirm({
                    message: lc('auto_scan_first_use'),
                    okButtonText: lc('enable'),
                    cancelButtonText: lc('disable')
                });
                if (!result && autoScan) {
                    toggleAutoScan(false);
                }
            }
            ApplicationSettings.setBoolean('showAutoScanWarning', false);
        }
        try {
            const nCropView = cropView.nativeView.nativeViewProtected;
            if (__ANDROID__) {
                const context = Utils.android.getApplicationContext();
                processor = new com.akylas.documentscanner.CustomImageAnalysisCallback(context, nCropView);
                cameraView.nativeView.processor = processor;
            } else {
                processor = OpencvDocumentProcessDelegate.alloc().initWithCropView(nCropView);
                cameraView.nativeView.processor = processor;
            }
            DEV_LOG && console.log('applyProcessor', processor, previewResizeThreshold);
            applyAutoScan(autoScan);
            processor.previewResizeThreshold = previewResizeThreshold;
        } catch (error) {
            console.error(error, error.stack);
        }
    }

    function onFinishEditing() {
        toggleEditing();
    }

    function toggleEditing() {
        editing = !editing;
        if (torchEnabled) {
            forceTorchDisabled(editing);
        }
        if (editing) {
            stopPreview();
        } else {
            startPreview();
        }
    }

    function getFlashIcon(flashMode) {
        switch (flashMode) {
            case 0:
                return 'mdi-flash-off';
            case 1:
                return 'mdi-flash';
            case 2:
                return 'mdi-flash-auto';
            case 3:
                return 'mdi-flash-red-eye';
            case 4:
                return 'mdi-flashlight';
        }
    }

    const borderStroke = 3;
    const borderPaint = new Paint();
    borderPaint.strokeWidth = borderStroke;
    borderPaint.style = Style.STROKE;
    borderPaint.color = 'white';

    function onAutoScanChanged(value) {
        if (takPictureBtnCanvas?.nativeView) {
            takPictureBtnCanvas.nativeView.invalidate();
        }
    }
    $: onAutoScanChanged(autoScan);
    function drawTakePictureBtnBorder({ canvas }: { canvas: Canvas }) {
        const w = canvas.getWidth();
        const h = canvas.getHeight();
        const radius = Math.min(w, h) / 2 - borderStroke / 2;
        if (autoScan) {
            canvas.drawArc(w / 2 - radius, h / 2 - radius, w / 2 + radius, h / 2 + radius, 0, 270, false, borderPaint);
        } else {
            canvas.drawCircle(w / 2, h / 2, radius, borderPaint);
        }
    }
    let cameraOpened = false;
    function onCameraOpen({ object }: { object: CameraView }) {
        if (__ANDROID__) {
            const currentResolution = cameraView.nativeView.getCurrentResolutionInfo();
            if (currentResolution) {
                cameraOptionsStore.update((state) => {
                    state['aspectRatio'] = currentResolution.aspectRatio;
                    state['pictureSize'] = currentResolution.pictureSize;
                    return state;
                });
            }
        }
        cameraOpened = true;
    }
</script>

<page bind:this={page} id="camera" actionBarHidden={true} statusBarStyle="dark" on:navigatedTo={onNavigatedTo} on:navigatedFrom={onNavigatedFrom}>
    <gridlayout backgroundColor="black" rows="auto,*,auto,auto">
        <cameraView
            bind:this={cameraView}
            {aspectRatio}
            autoFocus={true}
            captureMode={1}
            enablePinchZoom={true}
            flashMode={_actualFlashMode}
            iosCaptureMode="videoPhotoWithoutAudio"
            {pictureSize}
            rowSpan={viewsize === 'full' ? 4 : 2}
            {stretch}
            {zoom}
            on:cameraOpen={onCameraOpen}
            on:layoutChanged={onCameraLayoutChanged}
            on:loaded={applyProcessor}
            on:zoom={onZoom}
            on:tap={focusCamera} />
        <cropview bind:this={cropView} colors={[colorPrimary]} fillAlpha={120} isUserInteractionEnabled={false} rowSpan="2" strokeWidth={3} />
        <!-- <canvasView bind:this={canvasView} rowSpan="2" on:draw={onCanvasDraw} on:tap={focusCamera} /> -->
        <CActionBar backgroundColor="transparent" buttonsDefaultVisualState="black" modalWindow={true}>
            {#if startOnCam}
                <IconButton class="actionBarButton" defaultVisualState="black" text="mdi-image-plus" on:tap={showDocumentsList} />
                <IconButton class="actionBarButton" defaultVisualState="black" text="mdi-cogs" on:tap={() => showSettings()} />
            {/if}
        </CActionBar>

        <stacklayout horizontalAlignment="left" orientation="horizontal" row={2} verticalAlignment="center">
            <IconButton color="white" text={getFlashIcon(flashMode)} tooltip={lc('flash_mode')} on:tap={() => (flashMode = (flashMode + 1) % 4)} />
            <IconButton color="white" isSelected={torchEnabled} selectedColor={colorPrimary} text="mdi-flashlight" tooltip={lc('torch')} on:tap={switchTorch} />
            <IconButton color="white" text="mdi-camera-flip" tooltip={lc('toggle_camera')} on:tap={toggleCamera} />
        </stacklayout>
        {#if !startOnCam}
            <IconButton color="white" horizontalAlignment="right" isEnabled={cameraOpened} row={2} text="mdi-tune" on:tap={showCameraSettings} />
        {/if}

        <gridlayout columns="60,*,auto,*,60" ios:paddingBottom={30} android:marginBottom={30 + $navigationBarHeight} paddingTop={30} row={3}>
            <IconButton
                color="white"
                horizontalAlignment="left"
                marginLeft={16}
                text={batchMode ? 'mdi-image-multiple' : 'mdi-image'}
                tooltip={lc('batch_mode')}
                verticalAlignment="center"
                on:tap={() => (batchMode = !batchMode)} />

            <image
                bind:this={smallImageView}
                borderColor="white"
                col={1}
                ios:contextOptions={IMAGE_CONTEXT_OPTIONS}
                colorMatrix={getColorMatrix(colorType)}
                decodeWidth={Utils.layout.toDevicePixels(60)}
                height={60}
                horizontalAlignment="center"
                imageRotation={smallImageRotation}
                src={smallImage}
                stretch="aspectFit"
                verticalAlignment="center"
                width={60} />
            <gridlayout col={2} height={70} horizontalAlignment="center" opacity={takingPicture ? 0.6 : 1} verticalAlignment="center" width={70}>
                <canvasView bind:this={takPictureBtnCanvas} class:infinite-rotate={autoScan} on:draw={drawTakePictureBtnBorder}> </canvasView>
                <gridlayout backgroundColor={colorPrimary} borderRadius="50%" height={54} horizontalAlignment="center" width={54} on:tap={() => takePicture()} on:longPress={() => toggleAutoScan()} />
                <label color="white" fontSize={20} isUserInteractionEnabled={false} text={nbPages + ''} textAlignment="center" verticalAlignment="middle" visibility={nbPages ? 'visible' : 'hidden'} />
            </gridlayout>

            <IconButton
                col={4}
                color="white"
                horizontalAlignment="right"
                marginRight={16}
                text="mdi-check"
                tooltip={lc('finish')}
                verticalAlignment="center"
                visibility={canSaveDoc ? 'visible' : 'hidden'}
                on:tap={() => saveCurrentDocument()} />
        </gridlayout>

        <!-- <image
            bind:this={fullImageView}
            colorMatrix={getColorMatrix(colorType)}
            imageRotation={smallImageRotation}
            rowSpan={4}
            src={smallImage}
            stretch="aspectFit"
            visibility={showingFullScreenImage ? 'visible' : 'hidden'} /> -->
    </gridlayout>
</page>
