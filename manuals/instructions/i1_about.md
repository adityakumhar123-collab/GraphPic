Project description:
Graphpic is a software made for photographer where they bulk upload , event images and provide a qr or link to their clients. They open the link and click a pic of their face , and the application returns them all the photos in which their faces appear.

Key features:
- Bulk upload
- Multiple face detection in a single image
- Generate qr code for each event
- Mobile responsive UI
- browser based web app

Tech stack:
- react.js
- Node.js/Express.js
- mongodb
- qdrant
- Amazon s3 bucket
- redis queue
- insightface onnxruntime-gpu opencv-python numpy and arcface for face       extraction and embeddings generation.
 
Architecture:

- High-level overview:
we'll provide a web ui , for photographer to upload images , and in it they can create vaults for each event. Underneath though there will be a single bucket for all users and for all their vaults.

now, say a user bulk upload 3000 images , in a vault.
The ui will ,extract metadata and send only the metadata to the server , which will create post signed url for each image , and send it back to the ui , the ui will then use these url to upload the images to the bucket.
the bucket will be configured to sent a notification in the sqs queue , we'll have a polling service that will take the notification and push it to a redis queue. now the workers will fetch the uploaded images using the presigned url that got pushed with the notification , and the worker will process the image , exctract faces , create embeddings and store them in qdrant.

now the vaults will have a option to provide a link to the clients , say like clients/:id, where id is unique to the vault and limits the processing within that particular vault only . now the clients will click their photo and the ui will , perprocess it and send to the server , which will push the job to a queue again and put a pending document in the mongodb and return 200 ok. The worker will convert the photos to embeddings and perform a similarity search within that vault only using the metadata stored along with embeddings in qdrant,and will return all those images whose embeddings crosses a particular threshold , the worker will create presigned post url to temporarily store the result in s3 and update the status in mongo , the ui was meanwhile polling for those mongo success documents , once it gets those , it will query the server for the presigned url to fetch the images and display them to the clients and finally delete all the temporary data from s3 and mongodb.

- Semi low-level:
user Schema:
{
    _id:
    email:
    username:
    password:
    vault_id:{"event_name":_id} 
}

vault Schema:
{
    _id:
    username/event_id/unique_img_id:{metadata related to img}
    ...
}

Proposed Directory structure:
|package.json
|README.md
|pnpm-workspace.yaml
|config/
|packages/
|    |core/
|        |package.json
|        |db/
|        |services/
|apps/
|    |api/
|    |workers/
|    |poller/
|frontend/(react-vite app)
|manuals/
|    |docs/
|    |instructions/