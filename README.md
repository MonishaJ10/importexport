ImportMetadata.java
package com.example.importexport.model;

import jakarta.persistence.*;
import lombok.Data;

import java.sql.Timestamp;

@Data
@Entity
@Table(name = "import_metadata")
public class ImportMetadata {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String filename;
    private String service;

    @Column(name = "overwrite_flag")
    private char overwriteFlag;

    @Column(name = "uploaded_at", insertable = false, updatable = false)
    private Timestamp uploadedAt;
}



   
ImportMetadataRepository
package com.example.importexport.repository;

import com.example.importexport.model.ImportMetadata;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ImportMetadataRepository extends JpaRepository<ImportMetadata, Long> {
}



file import controller 
package com.example.importexport.controller;

import com.example.importexport.model.ImportMetadata;
import com.example.importexport.repository.ImportMetadataRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.*;
import java.util.List;

@RestController
@RequestMapping("/api/import")
@CrossOrigin
public class FileImportController {

    @Autowired
    private ImportMetadataRepository repository;

    private final String uploadDir = "/opt/uploads/";

    @PostMapping("/upload")
    public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file,
                                         @RequestParam("service") String service,
                                         @RequestParam("overwrite") boolean overwrite) {
        try {
            Path targetPath = Paths.get(uploadDir + file.getOriginalFilename());
            Files.write(targetPath, file.getBytes(), StandardOpenOption.CREATE);

            ImportMetadata metadata = new ImportMetadata();
            metadata.setFilename(file.getOriginalFilename());
            metadata.setService(service);
            metadata.setOverwriteFlag(overwrite ? 'Y' : 'N');
            repository.save(metadata);

            return ResponseEntity.ok("File uploaded and metadata saved.");
        } catch (IOException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Upload failed.");
        }
    }

    @GetMapping("/export-models")
    public ResponseEntity<List<ImportMetadata>> getExportModels() {
        return ResponseEntity.ok(repository.findAll());
    }

    @GetMapping("/json/{filename}")
    public ResponseEntity<String> getUploadedJson(@PathVariable String filename) {
        try {
            Path path = Paths.get(uploadDir + filename);
            if (!Files.exists(path)) {
                return ResponseEntity.status(HttpStatus.NOT_FOUND).body("File not found.");
            }
            String content = Files.readString(path);
            return ResponseEntity.ok(content);
        } catch (IOException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Failed to read file.");
        }
    }
}

















ts
import { Component, OnInit } from '@angular/core';
import { ColDef } from 'ag-grid-community';
import { HttpClient } from '@angular/common/http';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { AgGridModule } from 'ag-grid-angular';

@Component({
  selector: 'app-import-export-manager',
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css'],
  standalone: true,
  imports: [CommonModule, FormsModule, AgGridModule]
})
export class ImportExportManagerComponent implements OnInit {
  showImport = false;
  selectedService: string | null = null;
  overwrite = true;
  selectedFile: File | null = null;
  showWarning = false;
  previewJson: string | null = null;
  showModal = false;
  selectedRows: any[] = [];

  rowData: any[] = [];
  columnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'filename', headerName: 'Model' },
    { field: 'overwriteFlag', headerName: 'Mode' },
    { field: 'uploadedAt', headerName: 'Uploaded At' },
    { field: 'service', headerName: 'Service' },
    {
      headerName: 'Actions',
      cellRenderer: () => `<button class="preview-btn">Preview</button>`,
      width: 100,
    }
  ];

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.fetchExportModels();
  }

  onNewImportClick() {
    this.showImport = true;
    this.selectedService = null;
    this.selectedFile = null;
  }

  onServiceChange(event: Event) {
    const target = event.target as HTMLSelectElement;
    this.selectedService = target.value;
  }

  onFileSelected(event: any) {
    this.selectedFile = event.target.files[0];
  }

  upload() {
    if (!this.selectedFile || !this.selectedService) return;

    const formData = new FormData();
    formData.append('file', this.selectedFile);
    formData.append('service', this.selectedService);
    formData.append('overwrite', String(this.overwrite));

    this.http.post('/api/import/upload', formData).subscribe({
      next: () => {
        this.fetchExportModels();
        this.showImport = false;
        alert('Upload successful!');
      },
      error: () => {
        alert('Upload failed. Please check the file format or backend logs.');
      }
    });
  }

  fetchExportModels() {
    this.http.get<any[]>('/api/import/export-models').subscribe(data => {
      this.rowData = data;
    });
  }

  onSelectionChanged(event: any) {
    this.selectedRows = event.api.getSelectedRows();
  }

  exportModels() {
    if (this.selectedRows.length === 0) {
      this.showWarning = true;
    } else {
      this.showWarning = false;
      console.log('Exporting:', this.selectedRows);
      // Add download/export logic here if needed
    }
  }

  onGridReady(params: any) {
    params.api.addEventListener('cellClicked', (event: any) => {
      if (event.colDef.headerName === 'Actions' && event.event.target.classList.contains('preview-btn')) {
        this.loadJsonPreview(event.data.filename);
      }
    });
  }

  loadJsonPreview(filename: string) {
    this.http.get(`/api/import/json/${filename}`, { responseType: 'text' }).subscribe({
      next: (json) => {
        this.previewJson = json;
        this.showModal = true;
      },
      error: () => {
        alert('Failed to load JSON preview.');
      }
    });
  }

  closeModal() {
    this.showModal = false;
    this.previewJson = null;
  }
}



html
<div class="header">
  <span>Import-Export Manager</span>
  <button (click)="showImport = false">✖</button>
</div>

<div class="controls">
  <button (click)="onNewImportClick()">New Import</button>
  <button>Model</button>
</div>

<div *ngIf="showImport" class="import-form">
  <label>Service:
    <select (change)="onServiceChange($event)">
      <option value="">Select</option>
      <option value="Reconciliation">Reconciliation</option>
    </select>
  </label>

  <label>
    <input type="checkbox" [(ngModel)]="overwrite" /> Overwrite
  </label>

  <input type="file" (change)="onFileSelected($event)" />
  <button (click)="upload()">Upload</button>
</div>

<ag-grid-angular
  class="ag-theme-alpine"
  style="width: 100%; height: 300px;"
  [rowData]="rowData"
  [columnDefs]="columnDefs"
  rowSelection="multiple"
  (selectionChanged)="onSelectionChanged($event)"
  (gridReady)="onGridReady($event)">
</ag-grid-angular>

<button (click)="exportModels()">Export</button>

<div *ngIf="showWarning" class="warning">Please select one or more items to perform action.</div>

<!-- JSON Preview Modal -->
<div *ngIf="showModal" class="modal">
  <div class="modal-content">
    <h3>JSON Preview</h3>
    <pre>{{ previewJson }}</pre>
    <button (click)="closeModal()">Close</button>
  </div>
</div>




css
.preview-btn {
  background: #e0e0e0;
  border: 1px solid #999;
  padding: 4px 8px;
  cursor: pointer;
  border-radius: 4px;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  padding: 20px;
  max-width: 80%;
  max-height: 80%;
  overflow: auto;
  border-radius: 6px;
}







{
  "name": "ForeignExchangeRecon",
  "description": "Auto import of reconciliation model",
  "serviceName": "INTMATCH_3",
  "type": "RECONCILIATION",
  "fields": [
    {
      "name": "TransactionId",
      "type": "STRING",
      "isKey": true
    },
    {
      "name": "Amount",
      "type": "DECIMAL",
      "aggregation": "SUM"
    },
    {
      "name": "Currency",
      "type": "STRING"
    },
    {
      "name": "TradeDate",
      "type": "DATE"
    }
  ],
  "matchRules": [
    {
      "name": "MatchByTxnID",
      "priority": 1,
      "matchConditions": [
        {
          "leftField": "TransactionId",
          "operator": "EQUALS",
          "rightField": "TransactionId"
        }
      ]
    }
  ]
}







  