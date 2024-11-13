# Co-Viewer
// server/server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });


app.use(express.static('../client/build'));

let currentPage = 1;

io.on('connection', (socket) => {
  console.log('User connected', socket.id);

  
  socket.emit('page-update', currentPage);


  socket.on('change-page', (page) => {
    currentPage = page;
    io.emit('page-update', currentPage);
  });

  socket.on('disconnect', () => {
    console.log('User disconnected', socket.id);
  });
});


server.listen(3001, () => {
  console.log('Server running on port 3001');
});

// client/src/App.js
import React, { useEffect, useState } from 'react';
import { Viewer, Worker } from '@react-pdf-viewer/core';
import { defaultLayoutPlugin } from '@react-pdf-viewer/default-layout';
import io from 'socket.io-client';
import '@react-pdf-viewer/core/lib/styles/index.css';
import '@react-pdf-viewer/default-layout/lib/styles/index.css';

const socket = io('http://localhost:3001'); // Connect to the server

function App() {
  const defaultLayoutPluginInstance = defaultLayoutPlugin();
  const [currentPage, setCurrentPage] = useState(1);

  useEffect(() => {
    // Listen for page updates from the server
    socket.on('page-update', (page) => {
      setCurrentPage(page);
    });

    return () => {
      socket.off('page-update');
    };
  }, []);

  const handlePageChange = (page) => {
    setCurrentPage(page);
    socket.emit('change-page', page); // Send the new page to the server
  };

  return (
    <div className="App">
      <h1>PDF Co-Viewer</h1>
      <div style={{ marginBottom: '10px' }}>
        <button onClick={() => handlePageChange(currentPage - 1)} disabled={currentPage <= 1}>
          Previous Page
        </button>
        <span> Page {currentPage} </span>
        <button onClick={() => handlePageChange(currentPage + 1)}>
          Next Page
        </button>
      </div>
      <Worker workerUrl={`https://unpkg.com/pdfjs-dist@2.6.347/build/pdf.worker.min.js`}>
        <Viewer
          fileUrl="/sample.pdf" // Path to the PDF file
          plugins={[defaultLayoutPluginInstance]}
          initialPage={currentPage - 1}
          onPageChange={(e) => {
            setCurrentPage(e.currentPage + 1);
            socket.emit('change-page', e.currentPage + 1);
          }}
        />
      </Worker>
    </div>
  );
}

export default App;

