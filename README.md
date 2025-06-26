ts
import { Component, OnInit } from '@angular/core';
import { ColDef } from 'ag-grid-community';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-import-export-manager',
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css'],
  standalone: true,
})
export class ImportExportManagerComponent implements OnInit {
  showImport = false;
  selectedService: string | null = null;
  overwrite = true;
  selectedFile: File | null = null;
  showWarning = false;
  selectedRows: any[] = [];

  rowData: any[] = [];
  columnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'filename', headerName: 'Model' },
    { field: 'overwriteFlag', headerName: 'Mode' },
    { field: 'uploadedAt', headerName: 'Frequency' },
    { field: 'service', headerName: 'Service' },
    { field: 'status', headerName: 'Import Status' },
    {
      headerName: 'Actions',
      cellRenderer: (params: any) => {
        return `<button class="preview-btn">Preview</button>`;
      },
      width: 100,
    }
  ];

  previewJson: string | null = null;
  showModal = false;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.fetchExportModels();
  }

  onNewImportClick() {
    this.showImport = true;
    this.selectedService = null;
    this.selectedFile = null;
  }

  onServiceChange(service: string) {
    this.selectedService = service;
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
      error: (err) => {
        console.error('Upload failed', err);
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
      // Download/export logic here
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
    this.http.get(`/api/import/json/${filename}`, { responseType: 'text' }).subscribe(json => {
      this.previewJson = json;
      this.showModal = true;
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
    <select (change)="onServiceChange($event.target.value)">
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
  background: #ddd;
  border: none;
  padding: 4px 8px;
  cursor: pointer;
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
  border-radius: 4px;
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







  