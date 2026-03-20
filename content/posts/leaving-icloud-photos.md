+++
date = '2025-07-15T08:23:14+02:00'
title = 'Taking Back Control of My Photos with Immich'
+++

In order to reduce my reliance on cloud hosted services, I have made a shift to self-host as much as i can. This post serves as documentation for my latest addition: [Immich](https://immich.app/) as a iCloud Photos replacement.

I am going to assume you have a instance of immich running. If not, its well documented on their site.

## Getting the data from iCloud
First I needed to download all the photos/videos from iCloud. It is important to download the files with metadata embedded, as Immich unses that data to determine what date the picture or video was taken.
If you import a photo to your Photos app on iOS oder someone sends you a picture on WhatsApp, the metadata is often not embedded, thus immich would import it as todays date. To avoid this I used a tool called [icloudpd](https://github.com/icloud-photos-downloader/icloud_photos_downloader?tab=readme-ov-file) to download my full iCloud library. It uses docker to deploy a temporary container, so make sure you have that installed. By default it takes the date information from iCloud itself and creates a `YYYY/MM/DD` folder structure. To download my iCloud pictures i used the following command:
```bash
docker run -it --rm \
    --name icloudpd \
    -v ~/icloud-download:/data \
    -e TZ=Europe/Berlin \
    icloudpd/icloudpd:latest \
    icloudpd \
        --directory /data \
        --username user@domain.com \
        --watch-with-interval 3600
```
This downloads all my pictures and videos to `~/icloud-download`.

If you dont like to spin up a docker container, you can also use the `icloudpd` excutable. You can download it [here](github.com/icloud-photos-downloader/icloud_photos_downloader/releases/). Use the following syntax:
```bash
icloudpd --directory ~/icloud-download --username user@domain.com --watch-with-interval 3600
```
If you get en error like `ERROR Error processing user user@domain.de, continuing with next user...` it suggest a generic problem with the account. In most cases you just need to login to the web interface and accept the terms of use.

## Dealing with Live Photos
One small annoyance I had was Live Photos. It's a feature on iPhone, that takes a small video with the picture you have taken. I personally don't want those videos to be embedded into the pictures (alltough Immich supports that). Conveniently `icloudpd` splits Live Photos into a JPEG and a MOV file. I used this quick command to remove all the MOV files from the Live Photos:
```bash
find ~/icloud-download/* -type f -iname '*_HEVC.MOV' -delete
``` 

## Uploading to Immich
With the download and cleanup completed I used [immich-go](https://github.com/simulot/immich-go?tab=readme-ov-file) to upload the files to Immich. `immich-go` also uses the `YYYY/MM/DD` folder structure to determine the date, if no metadata data is found.
You need to create a API key for your Immich account to upload with `immich-go`. You can to that by clicking on your profile icon and navigating to `Account Settings > API Keys > New API Key`
I used the following command to upload:
```bash
./immich-go upload from-folder \
  --server=https://immich.domain.com \
  --api-key=1234567890 \
  ~/icloud-download \
  --pause-immich-jobs=FALSE \
  --log-level=DEBUG --api-trace --log-file=immich-go.log
```

Make sure to set a big enough quota for the user or dont set a quota at all, if you exceed the quot, `immich-go` will just exit with `ERR AssetUpload, POST, https://immich.domain.com/api/assets`.

Use the flags `--log-level=DEBUG --api-trace --log-file=immich-go.log` to write a persistant log and get verbose log output to troublehoot if someting doesnt work.

And that's it. Immich also has a great mobile app, wich reminds me of the pre iOS 18 UI in iCloud Photos.