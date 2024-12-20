# Project Plan

### Goal

The goal of this project is to build a function that will copy data and files from one Firebase project to another. The source data will be a JSON object identifying specific files as Firebase URLs. The JSON object will also contain links to images in Firebase storage as well as references to records from other files within the Firebase project. The copy function will:

1. Create a manifest of relevant links,
2. Copy the data,
3. Replace the source project links with new links from the target Firebase project.

### Steps

1. **Setup Google Cloud Function**
   - Create a Firebase Cloud Function using Firebase Admin SDK for accessing Firebase databases.
   - Set up Google Cloud functions and enable the necessary Firebase services.

2. **Create a Firebase Cloud Function with `firebase-admin` for database access.**
   - Initialize the Firebase Admin SDK with credentials for both Firebase A and Firebase B.

3. **Fetch Data**
   - Query all data from Firebase A's database using the Firebase Admin SDK.

4. **Resolve References**
   - Traverse through the projects in Firebase A and resolve references (e.g., `sectionMetadata`) using IDs to fetch related objects. This will also include resolving links to Firebase storage files.

5. **Copy Data**
   - Push the resolved data into Firebase B, maintaining the original structure. Also include Firebase Storage Data Transfer and then update the database in Firebase B with the new URLs for the uploaded files.

6. **Error Handling & Logs**
   - Implement robust error handling and logging to trace any issues during the transfer.

7. **Deploy the Function**
   - Deploy the Cloud Function to Google Cloud and trigger the function as required (e.g., via HTTP or Pub/Sub).

---

## Node.js Code

```javascript
const functions = require('firebase-functions');
const admin = require('firebase-admin');
const { Storage } = require('@google-cloud/storage');

admin.initializeApp(); // Firebase A Initialization

// Firebase B Initialization
const firebaseBConfig = {
    credential: admin.credential.cert(require('./firebaseB-credentials.json')),
    databaseURL: 'https://<firebaseB-project>.firebaseio.com',
};
const firebaseB = admin.initializeApp(firebaseBConfig, 'firebaseB');

const dbA = admin.database();
const dbB = firebaseB.database();
const storageA = admin.storage();
const storageB = new Storage({
    projectId: '<firebaseB-project>',
    keyFilename: './firebaseB-credentials.json',
});

// Copy all project items
exports.copyDataToFirebaseB = functions.https.onRequest(async (req, res) => {
    try {
        console.log("Starting data and storage copy...");

        // Fetch all data from Firebase A
        const dataSnapshot = await dbA.ref('/').once('value');
        const data = dataSnapshot.val();

        if (!data) {
            return res.status(404).send("No data found in Firebase A.");
        }

        // Process projects and resolve references including Firebase Storage files
        if (data.projects) {
            data.projects = await resolveProjectReferencesAndCopyStorage(data.projects, data);
        }

        // Write updated data to Firebase B
        await dbB.ref('/').set(data);

        console.log("Data and storage copy complete.");
        res.status(200).send("Data and storage successfully copied to Firebase B.");
    } catch (error) {
        console.error("Error copying data and storage:", error);
        res.status(500).send("Failed to copy data and storage.");
    }
});

// Copy specific project with specific id 
exports.copySpecificProjectToFirebaseB = functions.https.onRequest(async (req, res) => {
    const projectId = req.query.id; // Retrieve the project ID from query parameters

    if (!projectId) {
        return res.status(400).send("Project ID is required.");
    }

    try {
        console.log(`Starting data and storage copy for project ID: ${projectId}...`);

        // Fetch the specific project from Firebase A
        const projectSnapshot = await dbA.ref(`/projects/${projectId}`).once('value');
        const projectData = projectSnapshot.val();

        if (!projectData) {
            return res.status(404).send(`Project with ID ${projectId} not found in Firebase A.`);
        }

        // Process project data and resolve references, including storage files
        const resolvedProject = await resolveProjectReferencesAndCopyStorage(
            { [projectId]: projectData },
            await dbA.ref('/').once('value').then((snap) => snap.val())
        );

        // Write the specific project data to Firebase B
        await dbB.ref(`/projects/${projectId}`).set(resolvedProject[projectId]);

        console.log(`Project ID: ${projectId} successfully copied to Firebase B.`);
        res.status(200).send(`Project ID: ${projectId} successfully copied to Firebase B.`);
    } catch (error) {
        console.error("Error copying specific project:", error);
        res.status(500).send("Failed to copy specific project.");
    }
});


async function resolveProjectReferencesAndCopyStorage(projects, data) {
    const resolvedProjects = {};

    for (const [key, project] of Object.entries(projects)) {
        if (project.coverImageUrl) {
            // Copy the file from Firebase A Storage to Firebase B Storage
            const newCoverImageUrl = await copyFileToFirebaseB(project.coverImageUrl);
            project.coverImageUrl = newCoverImageUrl;
        }

        if (project.sectionsMetadata) {
            project.sectionsMetadata = await Promise.all(
                project.sectionsMetadata.map(async (section) => {
                    if (section.type === 'File' && section.id) {
                        // Resolve file reference and copy storage file if necessary
                        const fileReference = data.files[section.id];
                        if (fileReference && fileReference.storagePath) {
                            const newFileUrl = await copyFileToFirebaseB(fileReference.storagePath);
                            section.storagePath = newFileUrl;
                        }
                    }
                    return section;
                })
            );
        }
        resolvedProjects[key] = project;
    }

    return resolvedProjects;
}

async function copyFileToFirebaseB(fileUrl) {
    const sourceBucketName = 'firebaseA-project.appspot.com';
    const targetBucketName = 'firebaseB-project.appspot.com';

    const fileName = fileUrl.split('/o/')[1].split('?')[0]; // Extract file path from URL
    const sourceFile = storageA.bucket(sourceBucketName).file(fileName);
    const targetFile = storageB.bucket(targetBucketName).file(fileName);

    console.log(`Copying file: ${fileName} from ${sourceBucketName} to ${targetBucketName}`);

    // Download from Firebase A and upload to Firebase B
    await sourceFile.download({ destination: `/tmp/${fileName}` });
    await targetFile.upload(`/tmp/${fileName}`);

    // Get the new URL from Firebase B Storage
    const newFileUrl = `https://firebasestorage.googleapis.com/v0/b/${targetBucketName}/o/${encodeURIComponent(fileName)}?alt=media`;

    console.log(`File copied to Firebase B. New URL: ${newFileUrl}`);
    return newFileUrl;
}

```

## Finally

### For maintaining data integrity. We can add a validation to make sure that all the data are copied correctly, maintaining data structure and files are uploaded correctly

Below is a script to validate the storage and database transfers:

```javascript
const { Storage } = require('@google-cloud/storage');
const admin = require('firebase-admin');

admin.initializeApp(); // Firebase A Initialization

// Firebase B Initialization
const firebaseBConfig = {
    credential: admin.credential.cert(require('./firebaseB-credentials.json')),
    databaseURL: 'https://<firebaseB-project>.firebaseio.com',
};
const firebaseB = admin.initializeApp(firebaseBConfig, 'firebaseB');

const dbA = admin.database();
const dbB = firebaseB.database();
const storageA = admin.storage();
const storageB = new Storage({
    projectId: '<firebaseB-project>',
    keyFilename: './firebaseB-credentials.json',
});

async function validateTransfers() {
    try {
        console.log("Starting validation...");

        // Validate Database Transfer
        const dataA = (await dbA.ref('/').once('value')).val();
        const dataB = (await dbB.ref('/').once('value')).val();

        const databaseValidation = JSON.stringify(dataA) === JSON.stringify(dataB);
        console.log("Database Validation:", databaseValidation ? "Success" : "Failed");

        // Validate Storage Transfer
        const bucketA = storageA.bucket('firebaseA-project.appspot.com');
        const bucketB = storageB.bucket('firebaseB-project.appspot.com');

        const [filesA] = await bucketA.getFiles();
        const [filesB] = await bucketB.getFiles();

        const fileMapA = {};
        filesA.forEach(file => fileMapA[file.name] = file.metadata.size);

        const fileMapB = {};
        filesB.forEach(file => fileMapB[file.name] = file.metadata.size);

        const storageValidation = Object.keys(fileMapA).every(
            fileName => fileMapA[fileName] === fileMapB[fileName]
        );
        console.log("Storage Validation:", storageValidation ? "Success" : "Failed");

        // Output validation result
        if (databaseValidation && storageValidation) {
            console.log("Validation passed: All data and files transferred successfully.");
        } else {
            console.log("Validation failed: Check logs for discrepancies.");
        }
    } catch (error) {
        console.error("Error during validation:", error);
    }
}

// Run validation
validateTransfers();
```

### The script will do the following

Database Comparison:

- Fetch entire data trees from both Firebase A and Firebase B.
Perform a JSON deep comparison of the two datasets.

Storage Comparison:

- List all files from both Firebase A's and Firebase B's storage buckets.
Compare file names and sizes to ensure all files are accounted for.

Validation Logs:

- Log success or failure for database and storage validations.
Print details of any discrepancies for manual inspection.***

## Nice to have

### In case that there's a data descrepancy. it is nice to have a retry button or fix data feature. We would want to avoid doing the copy trasfer/copy process from the beginning to save costing and time. So we only want to fix the incorrect data

Below is a sample that does Enhanced Data Validation and Discrepancy Fixing:

```javascript
const crypto = require('crypto');
const fs = require('fs');

// Utility function to calculate MD5 checksum of a file
const calculateChecksum = (filePath) => {
    return new Promise((resolve, reject) => {
        const hash = crypto.createHash('md5');
        const stream = fs.createReadStream(filePath);

        stream.on('data', (data) => hash.update(data));
        stream.on('end', () => resolve(hash.digest('hex')));
        stream.on('error', reject);
    });
};

const storageValidationWithFixing = async (firebaseAUrls, firebaseBUrls) => {
    const sourceBucket = storageA.bucket('firebaseA-project.appspot.com');
    const targetBucket = storageB.bucket('firebaseB-project.appspot.com');
    const discrepancies = [];

    for (const fileUrl of firebaseAUrls) {
        const fileName = fileUrl.split('/o/')[1].split('?')[0];

        const sourceFile = sourceBucket.file(fileName);
        const targetFile = targetBucket.file(fileName);

        console.log(`Validating file: ${fileName}`);

        // Check if the file exists in both storage buckets
        const [sourceExists] = await sourceFile.exists();
        const [targetExists] = await targetFile.exists();

        if (!sourceExists) {
            console.error(`Missing file in Firebase A: ${fileName}`);
            discrepancies.push({ fileName, reason: 'Missing in Firebase A' });
            continue;
        }

        if (!targetExists) {
            console.warn(`Missing file in Firebase B: ${fileName}. Retrying transfer...`);
            await reTransferFile(sourceFile, targetFile, fileName);
            discrepancies.push({ fileName, reason: 'Transferred to Firebase B' });
            continue;
        }

        // Download files for checksum validation
        const tempSourcePath = `/tmp/source_${fileName}`;
        const tempTargetPath = `/tmp/target_${fileName}`;
        await sourceFile.download({ destination: tempSourcePath });
        await targetFile.download({ destination: tempTargetPath });

        const sourceChecksum = await calculateChecksum(tempSourcePath);
        const targetChecksum = await calculateChecksum(tempTargetPath);

        if (sourceChecksum !== targetChecksum) {
            console.warn(`Checksum mismatch for ${fileName}. Retrying transfer...`);
            await reTransferFile(sourceFile, targetFile, fileName);
            discrepancies.push({ fileName, reason: 'Checksum mismatch fixed' });
        } else {
            console.log(`File validated successfully: ${fileName}`);
        }

        // Cleanup temporary files
        fs.unlinkSync(tempSourcePath);
        fs.unlinkSync(tempTargetPath);
    }

    return discrepancies;
};

const reTransferFile = async (sourceFile, targetFile, fileName) => {
    console.log(`Re-transferring file: ${fileName}`);
    const tempPath = `/tmp/${fileName}`;
    await sourceFile.download({ destination: tempPath });
    await targetFile.upload(tempPath);
    fs.unlinkSync(tempPath);
};

// Combined validation and fixing script
exports.validateAndFixTransfer = functions.https.onRequest(async (req, res) => {
    try {
        // Fetch data from both databases
        const dataA = (await dbA.ref('/').once('value')).val();
        const dataB = (await dbB.ref('/').once('value')).val();

        // Extract file URLs
        const firebaseAUrls = extractFileUrls(dataA);
        const firebaseBUrls = extractFileUrls(dataB);

        // Validate storage with checksum and fixing
        const storageDiscrepancies = await storageValidationWithFixing(firebaseAUrls, firebaseBUrls);

        // Validate database content
        const databaseDiscrepancies = [];
        if (JSON.stringify(dataA) !== JSON.stringify(dataB)) {
            console.warn("Database content mismatch detected.");
            databaseDiscrepancies.push({ reason: 'Database mismatch' });
            await dbB.ref('/').set(dataA); // Re-sync data from A to B
            databaseDiscrepancies.push({ reason: 'Database re-synced from Firebase A' });
        }

        console.log("Validation completed with discrepancies:", {
            storage: storageDiscrepancies,
            database: databaseDiscrepancies,
        });

        res.status(200).send({
            message: "Validation and fixing completed.",
            storageDiscrepancies,
            databaseDiscrepancies,
        });
    } catch (error) {
        console.error("Error during validation and fixing:", error);
        res.status(500).send("Error during validation and fixing.");
    }
});

// Helper function to extract file URLs from database
function extractFileUrls(data) {
    const urls = [];
    const traverse = (obj) => {
        for (const key in obj) {
            if (typeof obj[key] === 'object') {
                traverse(obj[key]);
            } else if (key.toLowerCase().includes('url') && typeof obj[key] === 'string') {
                urls.push(obj[key]);
            }
        }
    };
    traverse(data);
    return urls;
}
```
