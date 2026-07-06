# Kontent Zavod — AWS S3 ga ko'chirish yo'riqnomasi

## Nima o'zgaradi?
- Supabase → AWS S3 (fayllar va ma'lumotlar uchun)
- Boshqa hech narsa o'zgarmaydi — ilova xuddi o'sha

---

## 1-qadam — AWS akkaunt
1. aws.amazon.com → "Create account" (yoki kiring)
2. Credit card kerak bo'ladi (lekin bepul tier bor: 5GB S3 bepul)

---

## 2-qadam — S3 Bucket yarating
1. AWS Console → S3 → "Create bucket"
2. Bucket name: `kontent-zavod-files`
3. Region: `eu-central-1` (Frankfurt — O'zbekistonga yaqin)
4. **"Block all public access"** — OLIB TASHLANG (uncheck) barcha 4 ta
5. → Create bucket

---

## 3-qadam — CORS sozlash
Bucket → Permissions → CORS → Edit → paste qiling:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": ["ETag"]
  }
]
```

---

## 4-qadam — Bucket Policy (public read)
Bucket → Permissions → Bucket policy → paste qiling
(bucket nomini o'zingiznikiga o'zgartiring):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::kontent-zavod-files/*"
    }
  ]
}
```

---

## 5-qadam — IAM User (API kalitlar)
1. AWS Console → IAM → Users → "Create user"
2. Nom: `kz-app-user`
3. Permissions → "Attach policies directly" → `AmazonS3FullAccess`
4. → Create user
5. User'ga kiring → Security credentials → "Create access key"
6. **Access key ID** va **Secret access key** ni yozib oling

---

## 6-qadam — AWS SDK o'rnatish
`index.html` ning `<head>` qismiga qo'shing:
```html
<script src="https://sdk.amazonaws.com/js/aws-sdk-2.1472.0.min.js"></script>
```

---

## 7-qadam — index.html da konfiguratsiya
Fayldagi `SUPABASE_URL` va `SUPABASE_KEY` qatorlarini mana bular bilan almashtiring:

```javascript
// AWS CONFIG — o'zingiznikini kiriting
const AWS_BUCKET = 'kontent-zavod-files';
const AWS_REGION = 'eu-central-1';
const AWS_ACCESS_KEY = 'AKIA...'; // IAM dan
const AWS_SECRET_KEY = 'xxx...';  // IAM dan

AWS.config.update({
  accessKeyId: AWS_ACCESS_KEY,
  secretAccessKey: AWS_SECRET_KEY,
  region: AWS_REGION
});
const s3 = new AWS.S3();

// Fayl yuklash
async function uploadFileAWS(file, progressId) {
  const key = 'uploads/' + Date.now() + '_' + file.name.replace(/[^a-zA-Z0-9._-]/g, '_');
  const params = {
    Bucket: AWS_BUCKET,
    Key: key,
    Body: file,
    ContentType: file.type,
  };
  const upload = s3.upload(params);
  // progress tracking
  upload.on('httpUploadProgress', (e) => {
    const pct = Math.round(e.loaded / e.total * 100);
    const bar = document.getElementById(progressId + '-bar');
    if (bar) bar.style.width = pct + '%';
  });
  const data = await upload.promise();
  return {
    path: key,
    url: `https://${AWS_BUCKET}.s3.${AWS_REGION}.amazonaws.com/${key}`,
    name: file.name,
    size: file.size
  };
}

// JSON data (kz_data.json) saqlash
async function saveDataAWS(payload) {
  await s3.putObject({
    Bucket: AWS_BUCKET,
    Key: 'kz_data.json',
    Body: JSON.stringify(payload),
    ContentType: 'application/json',
  }).promise();
}

// JSON data o'qish
async function loadDataAWS() {
  try {
    const data = await s3.getObject({
      Bucket: AWS_BUCKET, Key: 'kz_data.json'
    }).promise();
    return JSON.parse(data.Body.toString());
  } catch (e) {
    if (e.code === 'NoSuchKey') return null;
    throw e;
  }
}
```

---

## Narx
- **S3 Storage**: $0.023 per GB/oy → 10GB = $0.23/oy (~3000 so'm)
- **S3 Transfer**: $0.09 per GB (download) → Birinchi 100GB bepul
- **Jami**: taxminan $1-3/oy (Supabase Pro $25/oy dan arzonroq!)

---

## Xulosa
Agar AWS ni sozlashda yordam kerak bo'lsa — menga aytasiz,
men index.html ni to'liq AWS versiyasiga o'tkazib beraman.
