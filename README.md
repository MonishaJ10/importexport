import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { FormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatSelectModule } from '@angular/material/select';
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { ColDef } from 'ag-grid-community';
import { AgGridModule } from 'ag-grid-angular';
import { ImportExportService } from './import-export.service';

@Component({
  selector: 'app-import-export-manager',
  standalone: true,
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css'],
  imports: [
    CommonModule,
    FormsModule,
    MatCheckboxModule,
    MatFormFieldModule,
    MatSelectModule,
    MatButtonModule,
    MatIconModule,
    AgGridModule
  ]
})
export class ImportExportManagerComponent implements OnInit {
  selectedTab: 'import' | 'export' = 'import';
  services: string[] = [];
  selectedService = '';
  overwrite = false;
  selectedFile: File | null = null;
  rowData: any[] = [];
  selectedRows: any[] = [];
  showWarning = false;
  showModal = false;
  previewJson: string | null = null;
  showModelTable = false;

  columnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'filename', headerName: 'Model' },
    { field: 'overwriteFlag', headerName: 'Mode' },
    { field: 'uploadedAt', headerName: 'Uploaded At' },
    { field: 'service', headerName: 'Service' },
    {
      headerName: 'Actions',
      cellRenderer: () => '<button class="preview-btn">Preview</button>',
      width: 100
    }
  ];

  constructor(private service: ImportExportService) {}

  ngOnInit(): void {
    this.fetchServices();
    this.fetchExportModels(); // Optional: preload models
  }

  selectTab(tab: 'import' | 'export') {
    this.selectedTab = tab;
    if (tab === 'export') {
      this.fetchExportModels();
    }
  }

  fetchServices() {
    this.service.getServices().subscribe({
      next: (data) => {
        this.services = data;
      },
      error: () => alert('Failed to load services.')
    });
  }

  onFileSelected(event: any) {
    this.selectedFile = event.target.files[0];
  }

  upload() {
    if (!this.selectedFile || !this.selectedService) {
      alert('Please select a service and a file.');
      return;
    }

    const formData = new FormData();
    formData.append('file', this.selectedFile);
    formData.append('service', this.selectedService);
    formData.append('overwrite', String(this.overwrite));

    this.service.upload(formData).subscribe({
      next: () => {
        alert('Upload successful!');
        this.fetchExportModels();
        this.selectedFile = null;
        this.showModelTable = true;
      },
      error: () => alert('Upload failed.')
    });
  }

  fetchExportModels() {
    this.service.getExportModels().subscribe((data) => {
      this.rowData = data;
    });
  }

  onGridReady(params: any) {
    params.api.addEventListener('cellClicked', (event: any) => {
      if (
        event.colDef.headerName === 'Actions' &&
        event.event.target.classList.contains('preview-btn')
      ) {
        this.loadJsonPreview(event.data.filename);
      }
    });
  }

  loadJsonPreview(filename: string) {
    this.service.getJsonPreview(filename).subscribe({
      next: (json) => {
        this.previewJson = json;
        this.showModal = true;
      },
      error: () => alert('Failed to load JSON preview.')
    });
  }

  closeModal() {
    this.showModal = false;
    this.previewJson = null;
  }
}



<div class="manager-container">
  <h2>Import-Export Manager</h2>

  <div class="top-bar">
    <button mat-raised-button color="primary" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon> New Import
    </button>

    <button mat-raised-button [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
      <mat-icon>sync_alt</mat-icon> Model
    </button>

    <mat-form-field appearance="outline" class="dropdown">
      <mat-label>Select Recon Service</mat-label>
      <mat-select [(ngModel)]="selectedService">
        <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
      </mat-select>
    </mat-form-field>
  </div>

  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" hidden #fileInput (change)="onFileSelected($event)" />
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button color="accent" [disabled]="!selectedFile || !selectedService" (click)="upload()">
      Upload
    </button>

    <!-- ✅ Show AG Grid table below after upload -->
    <div *ngIf="showModelTable" class="import-models-table">
      <ag-grid-angular
        class="ag-theme-alpine"
        style="width: 100%; height: 300px;"
        [rowData]="rowData"
        [columnDefs]="columnDefs"
        rowSelection="multiple"
        (gridReady)="onGridReady($event)">
      </ag-grid-angular>
    </div>
  </div>

  <div *ngIf="selectedTab === 'export'" class="export-section">
    <!-- your Export section grid remains untouched -->
  </div>

  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>






















import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { FormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatSelectModule } from '@angular/material/select';
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { AgGridModule } from 'ag-grid-angular';
import { ColDef } from 'ag-grid-community';
import { ImportExportService } from './import-export.service';

@Component({
  selector: 'app-import-export-manager',
  standalone: true,
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css'],
  imports: [
    CommonModule,
    FormsModule,
    MatCheckboxModule,
    MatFormFieldModule,
    MatSelectModule,
    MatButtonModule,
    MatIconModule,
    AgGridModule
  ]
})
export class ImportExportManagerComponent implements OnInit {
  selectedTab: 'import' | 'export' = 'import';

  services: string[] = [];
  selectedService = '';
  overwrite = false;
  selectedFile: File | null = null;

  rowData: any[] = [];
  selectedRows: any[] = [];

  showWarning = false;
  showModal = false;
  previewJson: string | null = null;

  columnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'filename', headerName: 'Model' },
    { field: 'overwriteFlag', headerName: 'Mode' },
    { field: 'uploadedAt', headerName: 'Uploaded At' },
    { field: 'service', headerName: 'Service' },
    {
      headerName: 'Actions',
      cellRenderer: () => '<button class="preview-btn">Preview</button>',
      width: 100
    }
  ];

  constructor(private service: ImportExportService) {}

  ngOnInit(): void {
    this.fetchServices();
    this.fetchExportModels();
  }

  selectTab(tab: 'import' | 'export'): void {
    this.selectedTab = tab;
    if (tab === 'export') {
      this.fetchExportModels();
    }
  }

  fetchServices(): void {
    console.log('Fetching services...');
    this.service.getServices().subscribe({
      next: (data) => {
        console.log('Fetched services:', data);
        this.services = data;
      },
      error: () => alert('Failed to load services.')
    });
  }

  onFileSelected(event: any): void {
    this.selectedFile = event.target.files[0];
  }

  upload(): void {
    if (!this.selectedFile || !this.selectedService) {
      alert('Please select a service and a file.');
      return;
    }

    const formData = new FormData();
    formData.append('file', this.selectedFile);
    formData.append('service', this.selectedService);
    formData.append('overwrite', String(this.overwrite));

    this.service.upload(formData).subscribe({
      next: (response) => {
        alert('Upload successful!');
        this.fetchExportModels();
        this.selectedFile = null;
      },
      error: (error) => {
        console.error('Upload error:', error);
        alert('Upload failed: ' + (error.error?.error || 'Unknown error'));
      }
    });
  }

  fetchExportModels(): void {
    this.service.getExportModels().subscribe(data => {
      this.rowData = data;
    });
  }

  exportModels(): void {
    if (this.selectedRows.length === 0) {
      this.showWarning = true;
    } else {
      this.showWarning = false;
      console.log('Exporting models:', this.selectedRows);
      // Export logic to be added later
    }
  }

  onGridReady(params: any): void {
    params.api.addEventListener('cellClicked', (event: any) => {
      if (
        event.colDef.headerName === 'Actions' &&
        event.event.target.classList.contains('preview-btn')
      ) {
        this.loadJsonPreview(event.data.filename);
      }
    });
  }

  loadJsonPreview(filename: string): void {
    this.service.getJsonPreview(filename).subscribe({
      next: (json) => {
        this.previewJson = json;
        this.showModal = true;
      },
      error: () => alert('Failed to load JSON preview.')
    });
  }

  closeModal(): void {
    this.showModal = false;
    this.previewJson = null;
  }

  onSelectionChanged(event: any): void {
    this.selectedRows = event.api.getSelectedRows();
  }
}






<div class="manager-container">
  <h2>Import-Export Manager</h2>

  <div class="top-bar">
    <button mat-raised-button color="primary" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon> New Import
    </button>

    <button mat-raised-button [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
      <mat-icon>sync_alt</mat-icon> Model
    </button>

    <mat-form-field appearance="outline" class="dropdown">
      <mat-label>Select Recon Service</mat-label>
      <mat-select [(ngModel)]="selectedService">
        <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
      </mat-select>
    </mat-form-field>
  </div>

  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" hidden #fileInput (change)="onFileSelected($event)" />
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button color="accent" [disabled]="!selectedFile || !selectedService" (click)="upload()">Upload</button>
  </div>

  <div *ngIf="selectedTab === 'export'" class="export-section">
    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="rowData"
      [columnDefs]="columnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)"
      (gridReady)="onGridReady($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" (click)="exportModels()">Export</button>
    <div *ngIf="showWarning" class="warning">Please select one or more items to export.</div>
  </div>

  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>






















_________________________________________
onUpload(): void {
  if (!this.selectedFile || !this.selectedService) {
    alert('Please select a file and service.');
    return;
  }

  const formData = new FormData();
  formData.append('file', this.selectedFile);
  formData.append('service', this.selectedService);
  formData.append('overwrite', this.overwrite.toString());

  this.http.post<any>('http://localhost:8080/api/import/upload', formData).subscribe({
    next: (response) => {
      console.log('Upload success:', response);
      alert(response.message); // shows: "File uploaded and metadata saved."
    },
    error: (error) => {
      console.error('Upload error:', error);
      alert('Upload failed: ' + (error.error?.error || 'Unknown error'));
    }
  });
}









@PostMapping("/upload")
public ResponseEntity<Map<String, String>> upload(
        @RequestParam("file") MultipartFile file,
        @RequestParam("service") String service,
        @RequestParam("overwrite") boolean overwrite) {
    try {
        String uploadDir = System.getProperty("user.home") + "/Downloads/";
        Path targetPath = Paths.get(uploadDir + file.getOriginalFilename());

        Files.createDirectories(targetPath.getParent());
        Files.write(targetPath, file.getBytes(), StandardOpenOption.CREATE);

        // Save metadata
        ImportMetadata metadata = new ImportMetadata();
        metadata.setFilename(file.getOriginalFilename());
        metadata.setService(service);
        metadata.setOverwriteFlag(overwrite ? 'Y' : 'N');
        repository.save(metadata);

        Map<String, String> response = new HashMap<>();
        response.put("message", "File uploaded and metadata saved.");
        return ResponseEntity.ok(response);
    } catch (IOException e) {
        e.printStackTrace();
        Map<String, String> error = new HashMap<>();
        error.put("error", "Upload failed: " + e.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}



this.http.post<any>('http://localhost:8080/api/import/upload', formData).subscribe({
  next: (response) => {
    console.log('Upload success:', response);
    alert(response.message); // ✅ now response.message will work
  },
  error: (error) => {
    console.error('Upload error:', error);
    alert('Upload failed: ' + (error.error?.error || 'Unknown error'));
  }
});
















@PostMapping("/upload")
public ResponseEntity<?> upload(
        @RequestParam("file") MultipartFile file,
        @RequestParam("service") String service,
        @RequestParam("overwrite") boolean overwrite) {
    try {
        // Use Downloads folder (or change path if needed)
        String uploadDir = System.getProperty("user.home") + "/Downloads/";
        Path targetPath = Paths.get(uploadDir + file.getOriginalFilename());

        // Ensure the folder exists
        Files.createDirectories(targetPath.getParent());

        // Save the file
        Files.write(targetPath, file.getBytes(), StandardOpenOption.CREATE);

        // Save metadata
        ImportMetadata metadata = new ImportMetadata();
        metadata.setFilename(file.getOriginalFilename());
        metadata.setService(service);
        metadata.setOverwriteFlag(overwrite ? 'Y' : 'N');
        repository.save(metadata);

        // ✅ Return JSON response
        Map<String, String> response = new HashMap<>();
        response.put("message", "File uploaded and metadata saved.");
        return ResponseEntity.ok(response);

    } catch (IOException e) {
        e.printStackTrace();

        // ❌ Return error as JSON
        Map<String, String> error = new HashMap<>();
        error.put("error", "Upload failed: " + e.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}







upload() {
  if (!this.selectedFile || !this.selectedService) {
    alert('Please select a service and a file.');
    return;
  }

  // ✅ Optional but helps prevent ERR_UPLOAD_FILE_CHANGED
  const fileToUpload = this.selectedFile;

  const formData = new FormData();
  formData.append('file', fileToUpload);
  formData.append('service', this.selectedService);
  formData.append('overwrite', String(this.overwrite));

  this.service.upload(formData).subscribe({
    next: (res: any) => {
      alert(res); // or show success message
      this.fetchExportModels(); // Refresh the export list
      this.selectedFile = null;
    },
    error: (err) => {
      console.error('Upload error:', err); // 💡 Log error
      alert('Upload failed: ' + (err?.error || 'Unexpected error'));
    }
  });
}


<button mat-raised-button color="accent" [disabled]="!selectedFile || !selectedService" (click)="upload()">Upload</button>












@PostMapping("/upload")
public ResponseEntity<String> uploadFile(
        @RequestParam("file") MultipartFile file,
        @RequestParam("service") String service,
        @RequestParam("overwrite") String overwriteFlag) {

    try {
        File dir = new File(uploadDir);
        if (!dir.exists()) {
            dir.mkdirs();
        }

        String filePath = uploadDir + file.getOriginalFilename();
        file.transferTo(new File(filePath));

        System.out.println("File saved to: " + filePath);

        // Proceed to save metadata, parse JSON etc.
        return ResponseEntity.ok("Uploaded successfully");

    } catch (IOException e) {
        e.printStackTrace();
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Upload failed: " + e.getMessage());
    }
}









<div *ngIf="services.length === 0">No services loaded</div>
<div *ngIf="services.length > 0">
  Services loaded: {{ services | json }}
</div>









CREATE TABLE recon_services (
    id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name VARCHAR2(100) NOT NULL
);

INSERT INTO recon_services (name) VALUES ('Reconciliation');
INSERT INTO recon_services (name) VALUES ('CashManagement');
INSERT INTO recon_services (name) VALUES ('Securities');
COMMIT;

@Entity
@Table(name = "recon_services")
public class ReconService {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Getters & setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}

public interface ReconServiceRepository extends JpaRepository<ReconService, Long> {
    @Query("SELECT r.name FROM ReconService r")
    List<String> findAllServiceNames();
}

@RestController
@RequestMapping("/api/import")
public class FileImportController {

    @Autowired
    private ReconServiceRepository reconServiceRepository;

    @GetMapping("/services")
    public ResponseEntity<List<String>> getAvailableServices() {
        List<String> services = reconServiceRepository.findAllServiceNames();
        return ResponseEntity.ok(services);
    }
}












<div class="manager-container">
  <h2>Import-Export Manager</h2>

  <div class="top-bar">
    <button mat-raised-button color="primary" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon> New Import
    </button>

    <button mat-raised-button [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
      <mat-icon>sync_alt</mat-icon> Model
    </button>

    <mat-form-field appearance="outline" class="dropdown">
      <mat-label>Select Recon Service</mat-label>
      <mat-select [(ngModel)]="selectedService">
        <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
      </mat-select>
    </mat-form-field>
  </div>

  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" hidden #fileInput (change)="onFileSelected($event)" />
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button color="accent" (click)="upload()">Upload</button>
  </div>

  <div *ngIf="selectedTab === 'export'" class="export-section">
    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="rowData"
      [columnDefs]="columnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)"
      (gridReady)="onGridReady($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" (click)="exportModels()">Export</button>
    <div *ngIf="showWarning" class="warning">Please select one or more items to export.</div>
  </div>

  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>


.manager-container {
  padding: 20px;
}

.top-bar {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-bottom: 20px;
}

.dropdown {
  min-width: 200px;
}

.import-section, .export-section {
  margin-top: 20px;
}

.file-upload {
  margin-top: 15px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.warning {
  color: red;
  margin-top: 10px;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: white;
  padding: 20px;
  width: 500px;
  border-radius: 8px;
}















<div class="manager-container">
  <h2>Import-Export Manager</h2>

  <div class="top-bar">
    <div class="action-box" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon>
      <span>New Import</span>
    </div>

    <div class="action-box" [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
      <mat-icon>sync_alt</mat-icon>
      <span>Model</span>
    </div>

    <mat-form-field appearance="outline" class="dropdown">
      <mat-label>Select Recon Service</mat-label>
      <mat-select [(ngModel)]="selectedService">
        <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
      </mat-select>
    </mat-form-field>
  </div>

  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox class="overwrite" [(ngModel)]="overwrite">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" hidden #fileInput (change)="onFileSelected($event)" />
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button color="accent" (click)="upload()">Upload</button>
  </div>

  <div *ngIf="selectedTab === 'export'" class="export-section">
    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="rowData"
      [columnDefs]="columnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)"
      (gridReady)="onGridReady($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" (click)="exportModels()">Export</button>
    <div *ngIf="showWarning" class="warning">Please select one or more items to export.</div>
  </div>

  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>


.manager-container {
  padding: 20px;
  font-family: Arial, sans-serif;
}

.top-bar {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-bottom: 20px;
}

.action-box {
  border: 1px solid #ccc;
  padding: 12px 20px;
  text-align: center;
  cursor: pointer;
  border-radius: 5px;
  background-color: white;
  color: #007bff;
  display: flex;
  align-items: center;
  gap: 8px;
}

.action-box.active {
  background-color: #007bff;
  color: white;
}

.dropdown {
  width: 250px;
}

.overwrite {
  margin-bottom: 15px;
}

.file-upload {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 10px 0;
}

.warning {
  color: red;
  font-weight: bold;
  margin-top: 10px;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0,0,0,0.4);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background: white;
  padding: 20px;
  max-width: 600px;
  width: 90%;
  border-radius: 8px;
}













import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { FormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatSelectModule } from '@angular/material/select';
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { AgGridModule } from 'ag-grid-angular';
import { ImportExportService } from './import-export.service';

@Component({
  selector: 'app-import-export-manager',
  standalone: true,
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css'],
  imports: [
    CommonModule,
    FormsModule,
    MatCheckboxModule,
    MatFormFieldModule,
    MatSelectModule,
    MatButtonModule,
    MatIconModule,       // <-- ✅ This fixes the error
    AgGridModule
  ]
})
export class ImportExportManagerComponent implements OnInit {
  // ...your existing logic remains unchanged
}









<div class="manager-container">
  <h2>Import-Export Manager</h2>

  <div class="toggle-boxes">
    <div
      class="action-box"
      [class.active]="selectedTab === 'import'"
      (click)="selectTab('import')"
    >
      <mat-icon>cloud_upload</mat-icon>
      <span>New Import</span>
    </div>

    <div
      class="action-box"
      [class.active]="selectedTab === 'export'"
      (click)="selectTab('export')"
    >
      <mat-icon>sync_alt</mat-icon>
      <span>Model</span>
    </div>
  </div>

  <!-- Import UI -->
  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-form-field appearance="fill" class="dropdown">
      <mat-label>Select Recon Service</mat-label>
      <mat-select [(ngModel)]="selectedService">
        <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
      </mat-select>
    </mat-form-field>

    <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" hidden #fileInput (change)="onFileSelected($event)" />
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button color="accent" (click)="upload()">Upload</button>
  </div>

  <!-- Export UI -->
  <div *ngIf="selectedTab === 'export'" class="export-section">
    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="rowData"
      [columnDefs]="columnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)"
      (gridReady)="onGridReady($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" (click)="exportModels()">Export</button>
    <div *ngIf="showWarning" class="warning">Please select one or more items to export.</div>
  </div>

  <!-- JSON Preview Modal -->
  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>














import { Component, OnInit, ViewChild, ElementRef } from '@angular/core';
import { ColDef } from 'ag-grid-community';
import { HttpClient } from '@angular/common/http';
import { ImportExportService } from './import-export.service';

@Component({
  selector: 'app-import-export-manager',
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css']
})
export class ImportExportManagerComponent implements OnInit {
  selectedTab: 'import' | 'export' = 'import';
  services: string[] = [];
  selectedService = '';
  overwrite = false;
  selectedFile: File | null = null;

  rowData: any[] = [];
  selectedRows: any[] = [];
  showWarning = false;

  showModal = false;
  previewJson: string | null = null;

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

  constructor(private service: ImportExportService) {}

  ngOnInit(): void {
    this.fetchServices();
    this.fetchExportModels();
  }

  selectTab(tab: 'import' | 'export') {
    this.selectedTab = tab;
    if (tab === 'export') {
      this.fetchExportModels();
    }
  }

  fetchServices() {
    this.service.getServices().subscribe({
      next: (data) => (this.services = data),
      error: () => alert('Failed to load services.')
    });
  }

  onFileSelected(event: any) {
    this.selectedFile = event.target.files[0];
  }

  upload() {
    if (!this.selectedFile || !this.selectedService) {
      alert('Please select a service and a file.');
      return;
    }

    const formData = new FormData();
    formData.append('file', this.selectedFile);
    formData.append('service', this.selectedService);
    formData.append('overwrite', String(this.overwrite));

    this.service.upload(formData).subscribe({
      next: () => {
        alert('Upload successful!');
        this.fetchExportModels();
        this.selectedFile = null;
      },
      error: () => alert('Upload failed.')
    });
  }

  fetchExportModels() {
    this.service.getExportModels().subscribe(data => {
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
      console.log('Exporting models:', this.selectedRows);
    }
  }

  onGridReady(params: any) {
    params.api.addEventListener('cellClicked', (event: any) => {
      if (event.colDef.headerName === 'Actions' &&
          event.event.target.classList.contains('preview-btn')) {
        this.loadJsonPreview(event.data.filename);
      }
    });
  }

  loadJsonPreview(filename: string) {
    this.service.getJsonPreview(filename).subscribe({
      next: (json) => {
        this.previewJson = json;
        this.showModal = true;
      },
      error: () => alert('Failed to load JSON preview.')
    });
  }

  closeModal() {
    this.showModal = false;
    this.previewJson = null;
  }
}



.manager-container {
  font-family: Arial, sans-serif;
  padding: 20px;
}

.toggle-boxes {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
}

.action-box {
  border: 1px solid #ccc;
  padding: 20px;
  width: 150px;
  text-align: center;
  cursor: pointer;
  border-radius: 5px;
  transition: all 0.3s ease;
}

.action-box:hover,
.action-box.active {
  background-color: #0077b6;
  color: white;
  border-color: #0077b6;
}

.import-section,
.export-section {
  margin-top: 20px;
}

.dropdown {
  width: 300px;
  margin: 10px 0;
}

.file-upload {
  margin: 10px 0;
  display: flex;
  align-items: center;
  gap: 10px;
}

.warning {
  color: red;
  margin-top: 10px;
  font-weight: bold;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0,0,0,0.4);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background: white;
  padding: 20px;
  max-width: 600px;
  width: 90%;
  border-radius: 8px;
}




















@PostMapping("/upload")
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file,
                                     @RequestParam("service") String service,
                                     @RequestParam("overwrite") boolean overwrite) {
    try {
        // Use Downloads folder
        String uploadDir = System.getProperty("user.home") + "/Downloads/";
        Path targetPath = Paths.get(uploadDir + file.getOriginalFilename());

        // Ensure the folder exists
        Files.createDirectories(targetPath.getParent());

        // Save the file
        Files.write(targetPath, file.getBytes(), StandardOpenOption.CREATE);

        // Save metadata
        ImportMetadata metadata = new ImportMetadata();
        metadata.setFilename(file.getOriginalFilename());
        metadata.setService(service);
        metadata.setOverwriteFlag(overwrite ? 'Y' : 'N');
        repository.save(metadata);

        return ResponseEntity.ok("File uploaded and metadata saved.");
    } catch (IOException e) {
        e.printStackTrace();
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Upload failed: " + e.getMessage());
    }
}



System.out.println("File: " + file.getOriginalFilename());
System.out.println("Service: " + service);
System.out.println("Overwrite: " + overwrite);


@GetMapping("/services")
public ResponseEntity<List<String>> getAvailableServices() {
    List<String> services = List.of("Reconciliation", "CashManagement", "Securities"); // You can adjust these
    return ResponseEntity.ok(services);
}




html
<div class="import-export-container">
  <h2>Import-Export Manager</h2>

  <div class="tab-buttons">
    <button [class.active]="activeTab === 'import'" (click)="activeTab = 'import'">Import</button>
    <button [class.active]="activeTab === 'export'" (click)="activeTab = 'export'">Export</button>
  </div>

  <div *ngIf="activeTab === 'import'" class="import-section">
    <button (click)="showImportForm = !showImportForm">New Import</button>

    <div *ngIf="showImportForm" class="form-section">
      <label>Choose</label>
      <select class="dropdown" (change)="onServiceChange($event)">
        <option value="">Select Recon Service</option>
        <option *ngFor="let svc of services" [value]="svc">{{ svc }}</option>
      </select>

      <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox>

      <input type="file" (change)="onFileSelected($event)" />
      <button (click)="upload()">Upload</button>
    </div>
  </div>

  <div *ngIf="activeTab === 'export'" class="export-section">
    <button (click)="fetchExportModels()">Model</button>

    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px; margin-top: 20px;"
      [rowData]="rowData"
      [columnDefs]="columnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)"
      (gridReady)="onGridReady($event)">
    </ag-grid-angular>

    <button class="export-btn" (click)="exportModels()">Export</button>
    <div *ngIf="showWarning" class="warning">Please select one or more items to export.</div>
  </div>

  <!-- JSON Preview Modal -->
  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>

ts
import { Component, OnInit } from '@angular/core';
import { ColDef } from 'ag-grid-community';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { AgGridModule } from 'ag-grid-angular';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { ImportExportService } from './import-export.service';

@Component({
  selector: 'app-import-export-manager',
  standalone: true,
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css'],
  imports: [CommonModule, FormsModule, AgGridModule, MatCheckboxModule]
})
export class ImportExportManagerComponent implements OnInit {
  activeTab: 'import' | 'export' = 'import';
  showImportForm = false;
  overwrite = false;
  selectedService: string = '';
  selectedFile: File | null = null;

  services: string[] = [];
  rowData: any[] = [];
  selectedRows: any[] = [];

  showWarning = false;
  showModal = false;
  previewJson: string | null = null;

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

  constructor(private service: ImportExportService) {}

  ngOnInit(): void {
    this.fetchExportModels();
    this.fetchServices();
  }

  fetchServices() {
    this.service.getServices().subscribe({
      next: (data) => (this.services = data),
      error: () => alert('Failed to load service list.')
    });
  }

  onServiceChange(event: Event) {
    const target = event.target as HTMLSelectElement;
    this.selectedService = target.value;
  }

  onFileSelected(event: any) {
    this.selectedFile = event.target.files[0];
  }

  upload() {
    if (!this.selectedFile || !this.selectedService) {
      alert('Please select a service and a file.');
      return;
    }

    const formData = new FormData();
    formData.append('file', this.selectedFile);
    formData.append('service', this.selectedService);
    formData.append('overwrite', String(this.overwrite));

    this.service.upload(formData).subscribe({
      next: () => {
        this.fetchExportModels();
        this.showImportForm = false;
        this.selectedFile = null;
        alert('Upload successful!');
      },
      error: () => alert('Upload failed.')
    });
  }

  fetchExportModels() {
    this.service.getExportModels().subscribe(data => {
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
    }
  }

  onGridReady(params: any) {
    params.api.addEventListener('cellClicked', (event: any) => {
      if (event.colDef.headerName === 'Actions' &&
          event.event.target.classList.contains('preview-btn')) {
        this.loadJsonPreview(event.data.filename);
      }
    });
  }

  loadJsonPreview(filename: string) {
    this.service.getJsonPreview(filename).subscribe({
      next: (json) => {
        this.previewJson = json;
        this.showModal = true;
      },
      error: () => alert('Failed to load JSON preview.')
    });
  }

  closeModal() {
    this.showModal = false;
    this.previewJson = null;
  }
}


service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ImportExportService {
  constructor(private http: HttpClient) {}

  getServices(): Observable<string[]> {
    return this.http.get<string[]>('/api/import/services');
  }

  upload(formData: FormData): Observable<any> {
    return this.http.post('/api/import/upload', formData);
  }

  getExportModels(): Observable<any[]> {
    return this.http.get<any[]>('/api/import/export-models');
  }

  getJsonPreview(filename: string): Observable<string> {
    return this.http.get(`/api/import/json/${filename}`, { responseType: 'text' });
  }
}

css
.import-export-container {
  font-family: Arial, sans-serif;
  padding: 20px;
}

.tab-buttons button {
  margin-right: 10px;
  padding: 8px 16px;
  border: 1px solid #1976d2;
  background: white;
  color: #1976d2;
  cursor: pointer;
  border-radius: 4px;
}

.tab-buttons .active {
  background-color: #1976d2;
  color: white;
  font-weight: bold;
}

.form-section {
  margin-top: 15px;
  border: 1px solid #ccc;
  padding: 15px;
  border-radius: 8px;
}

.dropdown {
  width: 250px;
  padding: 5px;
  margin: 8px 0;
}

.warning {
  color: red;
  margin-top: 10px;
  font-weight: bold;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0,0,0,0.4);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background: white;
  padding: 20px;
  max-width: 600px;
  width: 90%;
  border-radius: 8px;
}






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







  
