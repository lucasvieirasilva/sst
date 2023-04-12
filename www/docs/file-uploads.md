---
title: File Uploads
description: "Add S3 file uploads to your SST app."
---

import HeadlineText from "@site/src/components/HeadlineText";

<HeadlineText>

A guide to adding S3 file uploads to your SST app.

</HeadlineText>

---

## Prerequisites

You'll need at least [Node.js 16](https://nodejs.org/) and [npm 7](https://www.npmjs.com/). You also need to have an AWS account and [**AWS credentials configured locally**](advanced/iam-credentials.md#loading-from-a-file).

---

## Create a new app

Start by creating a new SST + Next.js app by running the following command in your terminal.

```bash
npx create-sst@latest --template standard/nextjs
```

This will generate a new SST + Next.js app. Now to start your local environment, start SST.

```bash
npx sst dev
```

Then start Next.js in another terminal.

```bash
npm run dev
```

---

## Add an S3 bucket

Next, add an S3 bucket to your stacks. This will allow you to store the uploaded files.

```ts title="stacks/Default.ts"
const bucket = new Bucket(stack, "public", {
  cors: true,
});
```

The `cors` property enables Cross-Origin Resource Sharing (CORS), which allows your app to access the S3 bucket from your Next.js app.

---

## Bind the bucket

After adding the bucket, bind your Next.js app to it.

```diff title="stacks/Default.ts"
const site = new NextjsSite(stack, "site", {
  path: "packages/web",
+ bind: [bucket],
});
```

This allows Next.js app to access our S3 bucket.

---

## Generating a presigned URL

When a user uploads a file, we want to generate a presigned URL that allows them to upload the file directly to S3. We will do this using the AWS SDK.

```ts title="pages/index.ts"
export async function getServerSideProps() {
  const command = new PutObjectCommand({
    ACL: "public-read",
    Key: crypto.randomUUID(),
    Bucket: Bucket.public.bucketName,
  });
  const url = await getSignedUrl(new S3Client({}), command);

  return { props: { url } };
}
```

The above generates a presigned URL that allows `public-read` access to the uploaded files. You can change the ACL to `private` or `authenticated-read` if you prefer.

---

#### Add the imports

Import the required packages.

```ts title="pages/index.ts"
import crypto from "crypto";
import { Bucket } from "sst/node/bucket";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
```

Make sure to install them as well.

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

---

## Creating an upload form

Finally, we can create a form that allows users to upload a file:

```tsx title="pages/index.tsx"
export default function Home({ url }: { url: string }) {
  return (
    <main>
      <form
        onSubmit={async (e) => {
          e.preventDefault();

          const file = (e.target as HTMLFormElement).file.files?.[0]!;

          const image = await fetch(url, {
            body: file,
            method: "PUT",
            headers: {
              "Content-Type": file.type,
              "Content-Disposition": `attachment; filename="${file.name}"`,
            },
          });

          window.location.href = image.url.split("?")[0];
        }}
      >
        <input name="file" type="file" accept="image/png, image/jpeg" />
        <button type="submit">Upload</button>
      </form>
    </main>
  );
}
```

This form uploads the file directly to S3 and redirects to the file's URL.

---

## Deploy to prod

Let's end with deploying our app to production.

```bash
npx sst deploy --stage prod
```

---

That's it! You now know how to add file uploads with S3 to your SST app.