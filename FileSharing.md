# React - File Sharing

```javascript
import React, { useState, useEffect, useRef } from "react";
import toast, { Toaster } from "react-hot-toast";

export default function FileSharingApp() {
  const API_URL = import.meta.env.VITE_API_URL || "http://localhost:8080";

  const [file, setFile] = useState(null);
  const [files, setFiles] = useState([]);
  const [downloadName, setDownloadName] = useState("");
  const [isUploading, setIsUploading] = useState(false);
  const [isLoadingFiles, setIsLoadingFiles] = useState(false);
  const fileInputRef = useRef(null);

  // Fetch list of files -------------------------------------------------------------
  const fetchFiles = async () => {
    try {
      setIsLoadingFiles(true);
      const res = await fetch(`${API_URL}/files`);

      const data = res.ok ? await res.json() : [];

      // 3 second delay inline
      await new Promise((resolve) => setTimeout(resolve, 2000));

      setFiles(data);
    } catch (err) {
      console.error("List files error:", err);

      toast.error("Failed to fetch files");
      setFiles([]);
    } finally {
      setIsLoadingFiles(false);
    }
  };

// Call the fetchfiles function when user visit on page ---------------------------------

  useEffect(() => {
    fetchFiles();
  }, []);

  // Upload File -------------------------------------------------------------------------
  const uploadFile = async () => {

// sumbit without file
    if (!file) {
      toast.error("Select a file first!");
      return;
    }

// check the type of file
    const allowedTypes = ["image/png", "image/jpeg", "application/pdf"];
    if (!allowedTypes.includes(file.type)) {
      toast.error("Invalid file type! Allowed: PNG, JPG, PDF");
      return;
    }

//check the size of file
    if (file.size > 10 * 1024 * 1024) {
      toast.error("File too large (>10MB)");
      return;
    }

    setIsUploading(true);


  try {
      const formData = new FormData();
      formData.append("file", file);

      const res = await fetch(`${API_URL}/upload`, {
        method: "POST",
        body: formData,
      });

      const text = await res.text();

      if (res.ok) {
        toast.success(text || "Uploaded successfully!");
        setFile(null);
        if (fileInputRef.current) fileInputRef.current.value = "";
        fetchFiles();
      } else {
        toast.error(text || `Upload failed (status ${res.status})`);
      }
    } catch (err) {
      console.error("Upload error:", err);
      toast.error("Upload failed");
    } finally {
      setIsUploading(false);
    }
  };

  // Download File --------------------------------------------------------------
  const downloadFile = async (filename) => {
    if (!filename) filename = downloadName;
    if (!filename) {
      toast.error("Enter filename!");
      return;
    }
    try {
      const res = await fetch(
        `${API_URL}/download/${encodeURIComponent(filename)}`
      );
      if (!res.ok) {
        toast.error("File not found!");
        return;
      }
      const blob = await res.blob();
      const url = window.URL.createObjectURL(blob);
      const link = document.createElement("a");
      link.href = url;
      link.download = filename;
      link.click();
      window.URL.revokeObjectURL(url);
    } catch (err) {
      console.error("Download error:", err);
      toast.error("Download failed");
    }
  };

  // Delete File -----------------------------------------------------------------------------------
  const handleDelete = async (filename) => {
    if (!window.confirm(`Are you sure you want to delete "${filename}"?`))
      return;
    try {
      const res = await fetch(`${API_URL}/files/${filename}`, {
        method: "DELETE",
      });
      if (res.ok) {
        toast.success(`Deleted: ${filename}`);
        fetchFiles();
      } else {
        toast.error("Failed to delete file");
      }
    } catch (err) {
      console.error(err);
      toast.error("Error deleting file");
    }
  };

// UI Part ------------------------------------------------------------------------------------
  return (
    <main className="min-h-screen bg-gray-100 flex items-center justify-center p-6">
      <Toaster position="top-right" reverseOrder={false} />
      <div className="max-w-3xl w-full space-y-8">
        <h1 className="text-4xl font-bold text-center text-blue-700">
          📂 File Sharing App
        </h1>


 {/* --------------------------------- File Upload Section ---------------------------------- */}
        <div className="bg-white p-6 rounded-2xl shadow-md">
          <h2 className="text-xl font-semibold mb-4">Upload File</h2>
          <div className="flex items-center space-x-3">
            <input
              ref={fileInputRef}
              type="file"
              onChange={(e) => setFile(e.target.files?.[0] || null)}
              className="block w-full border rounded-lg p-2"
              aria-label="Choose file to upload"
              disabled={isUploading}
            />
            <button
              onClick={uploadFile}
              disabled={isUploading || !file}
              className={`px-4 py-2 rounded-lg text-white ${
                isUploading || !file
                  ? "bg-blue-300 cursor-not-allowed"
                  : "bg-blue-600 hover:bg-blue-700"
              }`}
            >
              {isUploading ? "Uploading…" : "Upload"}
            </button>
          </div>
        </div>


 {/* --------------------------------- File List Section ---------------------------------- */}

        <div className="bg-white p-6 rounded-2xl shadow-md">
          <div className="flex justify-between items-center mb-4">
            <h2 className="text-xl font-semibold">Available Files</h2>
            <button
              onClick={fetchFiles}
              className="px-3 py-1 bg-gray-700 text-white rounded-lg hover:bg-gray-800"
            >
              Refresh
            </button>
          </div>
          {isLoadingFiles ? (
            <p className="text-gray-500">Loading files...</p>
          ) : files.length === 0 ? (
            <p className="text-gray-500">No files available.</p>
          ) : (
            <ul className="space-y-2">
              {files.map((f, i) => (
                <li
                  key={i}
                  className="flex justify-between items-center bg-gray-50 p-3 rounded-lg border"
                >
                  <span className="truncate max-w-xs">{f}</span>
                  <div className="flex items-center space-x-2">
                    <button
                      onClick={() => downloadFile(f)}
                      className="px-3 py-1 bg-green-600 text-white rounded-lg hover:bg-green-700"
                    >
                      Download
                    </button>
                    <button
                      onClick={() => handleDelete(f)}
                      className="bg-red-500 text-white px-3 py-1 rounded hover:bg-red-600"
                    >
                      Delete
                    </button>
                  </div>
                </li>
              ))}
            </ul>
          )}
        </div>

    {/* --------------------------------- Download by Filename Section ---------------------------------- */}
      
        <div className="bg-white p-6 rounded-2xl shadow-md">
          <h2 className="text-xl font-semibold mb-4">Download by Filename</h2>
          <div className="flex items-center space-x-3">
            <input
              type="text"
              placeholder="Enter filename"
              value={downloadName}
              onChange={(e) => setDownloadName(e.target.value)}
              className="flex-1 border px-3 py-2 rounded-lg"
            />
            <button
              onClick={() => downloadFile()}
              className="px-4 py-2 bg-purple-600 text-white rounded-lg hover:bg-purple-700"
            >
              Download
            </button>
          </div>
        </div>
      </div>
    </main>
  );
}



```
