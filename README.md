model export service 

package com.example.work.service;

import com.example.work.dto.ExportModelDTO;
import com.example.work.dto.ReconModelRuleDTO;
import com.example.work.model.*;
import com.example.work.repository.*;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service
public class ModelExportService {

    @Autowired
    private ReconModelRepository reconModelRepository;

    @Autowired
    private FieldRepository fieldRepository;

    @Autowired
    private RuleRepository ruleRepository;

    @Autowired
    private ConditionRepository conditionRepository;

    // For POST /download
    public List<ExportModelDTO> getModelsByNames(List<String> modelNames) {
        List<ExportModelDTO> result = new ArrayList<>();

        for (String name : modelNames) {
            ReconModel model = reconModelRepository.findByName(name);
            if (model == null) continue;

            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(model.getName());
            dto.setDescription(model.getDescription());
            dto.setService(model.getService());
            dto.setContext(model.getContext());
            dto.setFrequency(model.getFrequency());
            dto.setModelMode(model.getModelMode());

            // Fields
            List<ReconModelField> fields = fieldRepository.findByModelName(name);
            dto.setFields(fields);

            // Rules & Conditions
            List<ReconModelRule> rules = ruleRepository.findByModelName(name);
            List<ReconModelRuleDTO> ruleDTOs = new ArrayList<>();

            for (ReconModelRule rule : rules) {
                ReconModelRuleDTO ruleDTO = new ReconModelRuleDTO();
                ruleDTO.setRuleName(rule.getRuleName());
                ruleDTO.setPriority(rule.getPriority());

                List<ReconRuleCondition> conditions = conditionRepository.findByRuleId(rule.getId());
                ruleDTO.setMatchConditions(conditions);

                ruleDTOs.add(ruleDTO);
            }

            dto.setMatchRules(ruleDTOs);
            result.add(dto);
        }

        return result;
    }

    // For GET /export-models
    public List<ExportModelDTO> getAllModels() {
        List<ReconModel> models = reconModelRepository.findAll();
        List<ExportModelDTO> result = new ArrayList<>();

        for (ReconModel model : models) {
            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(model.getName());
            dto.setDescription(model.getDescription());
            dto.setService(model.getService());
            dto.setContext(model.getContext());
            dto.setFrequency(model.getFrequency());
            dto.setModelMode(model.getModelMode());
            result.add(dto);
        }

        return result;
    }
}






@Service
public class ModelExportService {

    @Autowired
    private ReconModelRepository reconModelRepository;

    @Autowired
    private FieldRepository fieldRepository;

    @Autowired
    private RuleRepository ruleRepository;

    @Autowired
    private ConditionRepository conditionRepository;

    public List<ExportModelDTO> getModelsByNames(List<String> modelNames) {
        List<ExportModelDTO> result = new ArrayList<>();

        for (String name : modelNames) {
            ReconModel model = reconModelRepository.findByName(name);
            if (model == null) continue;

            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(model.getName());
            dto.setDescription(model.getDescription());
            dto.setService(model.getService());
            dto.setContext(model.getContext());
            dto.setFrequency(model.getFrequency());
            dto.setModelMode(model.getModelMode());

            // Fetch Fields
            List<ReconModelField> fields = fieldRepository.findByModelName(name);
            dto.setFields(fields);

            // Fetch Rules and nested Conditions
            List<ReconModelRule> rules = ruleRepository.findByModelName(name);
            List<ReconModelRuleDTO> ruleDTOs = new ArrayList<>();

            for (ReconModelRule rule : rules) {
                ReconModelRuleDTO ruleDTO = new ReconModelRuleDTO();
                ruleDTO.setRuleName(rule.getRuleName());
                ruleDTO.setPriority(rule.getPriority());

                List<ReconRuleCondition> conditions = conditionRepository.findByRuleId(rule.getId());
                ruleDTO.setMatchConditions(conditions);

                ruleDTOs.add(ruleDTO);
            }

            dto.setMatchRules(ruleDTOs);
            result.add(dto);
        }

        return result;
    }
}


model export controller 


@RestController
@RequestMapping("/api/export")
public class ModelExportController {

    @Autowired
    private ModelExportService exportService;

    @PostMapping("/download")
    public ResponseEntity<Resource> downloadModels(@RequestBody List<String> modelNames) throws IOException {
        List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);

        if (models.isEmpty()) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).body(null);
        }

        ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
        ZipOutputStream zos = new ZipOutputStream(zipOutStream);

        for (ExportModelDTO model : models) {
            String safeName = model.getName().replaceAll("[^a-zA-Z0-9_-]", "");
            String json = new ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(model);

            ZipEntry entry = new ZipEntry(safeName + ".json");
            zos.putNextEntry(entry);
            zos.write(json.getBytes(StandardCharsets.UTF_8));
            zos.closeEntry();
        }

        zos.finish();
        zos.close();

        ByteArrayResource resource = new ByteArrayResource(zipOutStream.toByteArray());

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip")
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .body(resource);
    }
}






package com.example.work.controller;

import com.example.work.dto.ExportModelDTO;
import com.example.work.service.EnhancedModelExportService;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.ByteArrayResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.util.List;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

@RestController
@RequestMapping("/api/import")
public class EnhancedModelExportController {

    @Autowired
    private EnhancedModelExportService enhancedModelExportService;

    @PostMapping("/export-models")
    public ResponseEntity<ByteArrayResource> exportSelectedModels(@RequestBody List<String> modelNames) throws IOException {
        List<ExportModelDTO> models = enhancedModelExportService.exportSelectedModels(modelNames);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ZipOutputStream zos = new ZipOutputStream(baos);
        ObjectMapper mapper = new ObjectMapper();

        for (ExportModelDTO model : models) {
            String json = mapper.writerWithDefaultPrettyPrinter().writeValueAsString(model);
            ZipEntry entry = new ZipEntry(model.getName().replaceAll("\\s+", "_") + ".json");
            zos.putNextEntry(entry);
            zos.write(json.getBytes());
            zos.closeEntry();
        }

        zos.close();

        ByteArrayResource resource = new ByteArrayResource(baos.toByteArray());

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip")
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .contentLength(resource.contentLength())
                .body(resource);
    }
}







package com.example.work.service;

import com.example.work.dto.ExportModelDTO;
import com.example.work.dto.ReconModelRuleDTO;
import com.example.work.model.*;
import com.example.work.repository.*;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service
public class EnhancedModelExportService {

    @Autowired
    private ReconModelRepository reconModelRepository;

    @Autowired
    private FieldRepository fieldRepository;

    @Autowired
    private RuleRepository ruleRepository;

    @Autowired
    private ConditionRepository conditionRepository;

    public List<ExportModelDTO> exportSelectedModels(List<String> modelNames) {
        List<ExportModelDTO> result = new ArrayList<>();

        for (String name : modelNames) {
            ReconModel model = reconModelRepository.findByName(name);
            if (model == null) continue;

            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(model.getName());
            dto.setDescription(model.getDescription());
            dto.setService(model.getService());
            dto.setContext(model.getContext());
            dto.setFrequency(model.getFrequency());
            dto.setModelMode(model.getModelMode());

            // Add fields
            List<ReconModelField> fields = fieldRepository.findByModelName(name);
            dto.setFields(fields);

            // Add rules + conditions
            List<ReconModelRule> rules = ruleRepository.findByModelName(name);
            List<ReconModelRuleDTO> ruleDTOs = new ArrayList<>();

            for (ReconModelRule rule : rules) {
                ReconModelRuleDTO ruleDTO = new ReconModelRuleDTO();
                ruleDTO.setRuleName(rule.getRuleName());
                ruleDTO.setPriority(rule.getPriority());

                List<ReconRuleCondition> conditions = conditionRepository.findByRuleId(rule.getId());
                ruleDTO.setMatchConditions(conditions);

                ruleDTOs.add(ruleDTO);
            }

            dto.setMatchRules(ruleDTOs);
            result.add(dto);
        }

        return result;
    }
}








package com.example.work.model;

import jakarta.persistence.*;
import lombok.Data;

@Entity
@Table(name = "recon_models")
@Data
public class ReconModel {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private String service;
    private String context;
    private String frequency;

    @Column(name = "model_mode")
    private String modelMode;
}






package com.example.work.service;

import com.example.work.dto.ExportModelDTO;
import com.example.work.dto.ReconModelRuleDTO;
import com.example.work.model.*;
import com.example.work.repository.*;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service
public class EnhancedModelExportService {

    @Autowired
    private ReconModelRepository reconModelRepository;

    @Autowired
    private FieldRepository fieldRepository;

    @Autowired
    private RuleRepository ruleRepository;

    @Autowired
    private ConditionRepository conditionRepository;

    public List<ExportModelDTO> exportSelectedModels(List<String> modelNames) {
        List<ExportModelDTO> result = new ArrayList<>();

        for (String name : modelNames) {
            ReconModel model = reconModelRepository.findByName(name);
            if (model == null) continue;

            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(model.getName());
            dto.setDescription(model.getDescription());
            dto.setService(model.getService());
            dto.setContext(model.getContext());
            dto.setFrequency(model.getFrequency());
            dto.setModelMode(model.getModelMode());

            List<ReconModelField> fields = fieldRepository.findByModelName(name);
            dto.setFields(fields);

            List<ReconModelRule> rules = ruleRepository.findByModelName(name);
            List<ReconModelRuleDTO> ruleDTOs = new ArrayList<>();

            for (ReconModelRule rule : rules) {
                ReconModelRuleDTO ruleDTO = new ReconModelRuleDTO();
                ruleDTO.setRuleName(rule.getRuleName());
                ruleDTO.setPriority(rule.getPriority());

                List<ReconRuleCondition> conditions = conditionRepository.findByRuleId(rule.getId());
                ruleDTO.setMatchConditions(conditions);

                ruleDTOs.add(ruleDTO);
            }

            dto.setMatchRules(ruleDTOs);
            result.add(dto);
        }

        return result;
    }
}










public interface FieldRepository extends JpaRepository<ReconModelField, Long> {
    List<ReconModelField> findByModelName(String modelName);
}

public interface RuleRepository extends JpaRepository<ReconModelRule, Long> {
    List<ReconModelRule> findByModelName(String modelName);
}

public interface ConditionRepository extends JpaRepository<ReconRuleCondition, Long> {
    List<ReconRuleCondition> findByRuleId(Long ruleId);
}





export model DTO 
@Data
public class ExportModelDTO {
    private String name;
    private String description;
    private String service;
    private String context;
    private String frequency;
    private String modelMode;
    private List<ReconModelField> fields;
    private List<ReconModelRuleDTO> matchRules;
}

recon model rule DTO
@Data
public class ReconModelRuleDTO {
    private String ruleName;
    private int priority;
    private List<ReconRuleCondition> matchConditions;
}

Entities
recon model rule.java
@Entity
@Table(name = "recon_model_rules")
@Data
public class ReconModelRule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String modelName;
    private String ruleName;
    private int priority;
}

recon model field.java
@Entity
@Table(name = "recon_model_fields")
@Data
public class ReconModelField {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String modelName;
    private String fieldName;
    private String fieldType;
    private String aggregation;
    private char isKey;
}

recon rule condition.java
@Entity
@Table(name = "recon_rule_conditions")
@Data
public class ReconRuleCondition {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long ruleId;
    private String leftField;
    private String operator;
    private String rightField;
}


-_---------------------------------
INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Switzerland_Settlement', 'SettlementId', 'STRING', 'Y', NULL);

INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Switzerland_Settlement', 'NetAmount', 'DECIMAL', 'N', 'SUM');

INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Switzerland_Settlement', 'Currency', 'STRING', 'N', NULL);

INSERT INTO recon_model_rules (model_name, rule_name, priority)
VALUES ('Switzerland_Settlement', 'MatchBySettlementId', 1);

SELECT id FROM recon_model_rules WHERE model_name = 'Switzerland_Settlement' AND rule_name = 'MatchBySettlementId';

INSERT INTO recon_rule_conditions (rule_id, left_field, operator, right_field)
VALUES (3, 'SettlementId', 'EQUALS', 'SettlementId');













INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Germany_Nostro', 'TransactionId', 'STRING', 'Y', NULL);

INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Germany_Nostro', 'Amount', 'DECIMAL', 'N', 'SUM');

INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Germany_Nostro', 'Currency', 'STRING', 'N', NULL);

INSERT INTO recon_model_rules (model_name, rule_name, priority)
VALUES ('Germany_Nostro', 'MatchByTxnID', 1);

SELECT id FROM recon_model_rules WHERE model_name = 'Germany_Nostro' AND rule_name = 'MatchByTxnID';

INSERT INTO recon_rule_conditions (rule_id, left_field, operator, right_field)
VALUES (2, 'TransactionId', 'EQUALS', 'TransactionId');














INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Germany_Holdings', 'TransactionId', 'STRING', 'Y', NULL);

INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Germany_Holdings', 'Amount', 'DECIMAL', 'N', 'SUM');

INSERT INTO recon_model_fields (model_name, field_name, field_type, is_key, aggregation)
VALUES ('Germany_Holdings', 'Currency', 'STRING', 'N', NULL);


INSERT INTO recon_model_rules (model_name, rule_name, priority)
VALUES ('Germany_Holdings', 'MatchByTxnID', 1);

SELECT id FROM recon_model_rules WHERE rule_name = 'MatchByTxnID' AND model_name = 'Germany_Holdings';

INSERT INTO recon_rule_conditions (rule_id, left_field, operator, right_field)
VALUES (1, 'TransactionId', 'EQUALS', 'TransactionId');








CREATE TABLE recon_model_fields (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  model_name VARCHAR2(100),
  field_name VARCHAR2(100),
  field_type VARCHAR2(50),
  is_key CHAR(1),
  aggregation VARCHAR2(50)
);

CREATE TABLE recon_model_rules (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  model_name VARCHAR2(100),
  rule_name VARCHAR2(100),
  priority NUMBER
);

CREATE TABLE recon_rule_conditions (
  id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  rule_id NUMBER,
  left_field VARCHAR2(100),
  operator VARCHAR2(50),
  right_field VARCHAR2(100),
  FOREIGN KEY (rule_id) REFERENCES recon_model_rules(id)
);

export model DTO
@Data
public class ExportModelDTO {
    private String name;
    private String description;
    private String service;
    private String context;
    private String frequency;
    private String modelMode;
    private List<ReconModelField> fields;
    private List<ReconModelRuleDTO> matchRules;
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

        ImportMetadata metadata = new ImportMetadata();
        metadata.setFilename(file.getOriginalFilename());
        metadata.setService(service);
        metadata.setOverwriterlag(overwrite ? 'Y' : 'N');
        metadata.setFileSize(file.getSize()); // ✅ Set file size in bytes
        metadata.setImportStatus("Success");  // ✅ Hardcoded for now — or set based on logic

        repository.save(metadata);

        Map<String, String> response = new HashMap<>();
        response.put("message", "File uploaded and metadata saved.");
        return ResponseEntity.ok(response);

    } catch (IOException e) {
        e.printStackTrace();

        // Save failed metadata if needed
        ImportMetadata metadata = new ImportMetadata();
        metadata.setFilename(file.getOriginalFilename());
        metadata.setService(service);
        metadata.setOverwriterlag(overwrite ? 'Y' : 'N');
        metadata.setImportStatus("Failure"); // ✅ Mark as failure

        repository.save(metadata);

        Map<String, String> error = new HashMap<>();
        error.put("error", "Upload failed: " + e.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}









import { Component, OnInit, ViewChild } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatSelectModule } from '@angular/material/select';
import { MatButtonModule } from '@angular/material/button';
import { MatIconModule } from '@angular/material/icon';
import { AgGridModule, ColDef } from 'ag-grid-angular';
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

  importRowData: any[] = [];
  exportRowData: any[] = [];
  selectedRows: any[] = [];

  showModal = false;
  previewJson: string | null = null;
  showModelTable = false;

  columnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'filename', headerName: 'Model' },
    { field: 'overwriteFlag', headerName: 'Mode' },
    { field: 'uploadedAt', headerName: 'Uploaded At' },
    { field: 'service', headerName: 'Service' },
    { field: 'fileSize', headerName: 'File Size (KB)', valueFormatter: this.formatSize },
    {
      field: 'importStatus',
      headerName: 'Status',
      cellRenderer: (params: any) => {
        const status = params.value;
        const color = status === 'Success' ? 'rgb(0,110,121)' : 'red';
        return `<span style="color: ${color}; font-weight: 600;">${status}</span>`;
      }
    },
    {
      headerName: 'Actions',
      cellRenderer: () => '<button class="preview-btn">Preview</button>',
      width: 100
    }
  ];

  exportColumnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'name', headerName: 'Model' },
    { field: 'model_mode', headerName: 'Mode' },
    { field: 'frequency', headerName: 'Frequency' },
    { field: 'context', headerName: 'Context' },
    { field: 'service', headerName: 'Service' }
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
      next: (data) => this.services = data,
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
        console.log('Upload successful');
        this.fetchImportTable();
        this.selectedFile = null;
        this.showModelTable = true;
      },
      error: (err) => {
        console.error('Upload failed:', err);
        alert('Upload failed.');
      }
    });
  }

  fetchImportTable() {
    this.service.getImportMetadata().subscribe({
      next: (data) => {
        console.log('Fetched import metadata:', data);
        this.importRowData = data;
      },
      error: (err) => {
        console.error('Failed to load import metadata:', err);
      }
    });
  }

  fetchExportModels() {
    this.service.getExportModels().subscribe({
      next: (data) => this.exportRowData = data,
      error: () => alert('Failed to load export models.')
    });
  }

  onSelectionChanged(event: any) {
    this.selectedRows = event.api.getSelectedRows();
  }

  exportSelectedModels() {
    const modelNames = this.selectedRows.map(row => row.name);
    if (modelNames.length === 0) {
      alert('No models selected for export.');
      return;
    }

    this.service.downloadModels(modelNames).subscribe(blob => {
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'models.zip';
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      window.URL.revokeObjectURL(url);
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

  get rowData() {
    return this.selectedTab === 'import' ? this.importRowData : this.exportRowData;
  }

  formatSize(params: any) {
    if (!params.value) return '';
    const sizeInKB = parseFloat(params.value) / 1024;
    return `${sizeInKB.toFixed(1)} KB`;
  }
}















ALTER TABLE import_metadata ADD file_size NUMBER;
ALTER TABLE import_metadata ADD import_status VARCHAR2(20);









@RestController
@RequestMapping("/api/import")
@CrossOrigin
public class FileImportController {

    @Autowired
    private ImportMetadataRepository repository;

    @Autowired
    private ReconserviceRepository reconserviceRepository;

    private final String uploadDir = System.getProperty("user.home") + "/Downloads/";

    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam("service") String service,
            @RequestParam("overwrite") boolean overwrite
    ) {
        Map<String, String> response = new HashMap<>();

        try {
            Path targetPath = Paths.get(uploadDir + file.getOriginalFilename());
            Files.createDirectories(targetPath.getParent());
            Files.write(targetPath, file.getBytes(), StandardOpenOption.CREATE);

            ImportMetadata metadata = new ImportMetadata();
            metadata.setFilename(file.getOriginalFilename());
            metadata.setService(service);
            metadata.setOverwriteFlag(overwrite ? 'Y' : 'N');
            metadata.setFileSize(file.getSize()); // <-- Set file size
            metadata.setImportStatus("Success");  // <-- Set status

            repository.save(metadata);

            response.put("message", "File uploaded and metadata saved.");
            return ResponseEntity.ok(response);
        } catch (IOException e) {
            e.printStackTrace();

            ImportMetadata metadata = new ImportMetadata();
            metadata.setFilename(file.getOriginalFilename());
            metadata.setService(service);
            metadata.setOverwriteFlag(overwrite ? 'Y' : 'N');
            metadata.setFileSize(file.getSize()); // still capture size
            metadata.setImportStatus("Failure");  // <-- Failed status

            repository.save(metadata);

            response.put("error", "Upload failed: " + e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
        }
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

    @GetMapping("/services")
    public ResponseEntity<List<String>> getAvailableServices() {
        List<String> services = reconserviceRepository.findAllServiceNames();
        return ResponseEntity.ok(services);
    }

    @GetMapping("/metadata")
    public ResponseEntity<List<ImportMetadata>> getAllImportMetadata() {
        List<ImportMetadata> list = repository.findAll(Sort.by(Sort.Direction.DESC, "uploadedAt"));
        return ResponseEntity.ok(list);
    }
}












:host ::ng-deep .mat-mdc-checkbox.mat-accent .mdc-checkbox__background {
  background-color: rgb(0, 110, 121) !important;
  border-color: rgb(0, 110, 121) !important;
}

:host ::ng-deep .mat-mdc-checkbox.mat-accent .mdc-checkbox__ripple {
  background-color: rgb(0, 110, 121) !important;
}

:host ::ng-deep .mat-mdc-checkbox.mat-accent .mdc-checkbox__checkmark-path {
  stroke: white !important;
}











:host ::ng-deep mat-checkbox.mat-accent .mat-checkbox-background {
  background-color: rgb(0, 110, 121) !important;
}

:host ::ng-deep mat-checkbox.mat-accent .mat-checkbox-frame {
  border-color: rgb(0, 110, 121) !important;
}

:host ::ng-deep mat-checkbox.mat-accent.mat-checkbox-checked .mat-checkbox-checkmark-path {
  stroke: white !important; /* Optional: makes checkmark visible */
}








$uinextgen-primary: mat.define-palette(mat.$indigo-palette);
$uinextgen-accent: mat.define-palette(mat.$teal-palette, A200, A100, A400); // ✅ Changed to teal
$uinextgen-warn: mat.define-palette(mat.$red-palette);

$uinextgen-theme: mat.define-light-theme((
  color: (
    primary: $uinextgen-primary,
    accent: $uinextgen-accent,
    warn: $uinextgen-warn,
  ),
  typography: mat.define-typography-config(),
  density: 0
));






.manager-container {
  max-width: 900px;         /* limit width */
  margin: 0 auto;           /* center it */
  padding: 16px;            /* spacing inside */
  font-size: 14px;          /* reduce font size */
  transform: scale(0.95);   /* slightly scale down everything */
}








::ng-deep .import-export-manager-container .mat-checkbox-checked.mat-accent .mat-checkbox-background {
  background-color: rgb(0, 110, 121) !important;
}

::ng-deep .import-export-manager-container .mat-checkbox.mat-accent .mat-checkbox-frame {
  border-color: rgb(0, 110, 121) !important;
}

::ng-deep .import-export-manager-container .mat-checkbox-checked .mat-checkbox-layout .mat-checkbox-label {
  color: rgb(0, 110, 121) !important;
}








/* Checkbox fill color */
.mat-checkbox-checked.mat-accent .mat-checkbox-background,
.mat-checkbox-indeterminate.mat-accent .mat-checkbox-background {
  background-color: rgb(0, 110, 121) !important;
}

/* Checkbox outline color */
.mat-checkbox.mat-accent .mat-checkbox-frame {
  border-color: rgb(0, 110, 121) !important;
}

/* Checkbox label color */
.mat-checkbox-checked.mat-accent .mat-checkbox-layout .mat-checkbox-label {
  color: rgb(0, 110, 121) !important;
}

/* Form field border */
.mat-form-field-appearance-outline .mat-form-field-outline {
  border-color: rgb(0, 110, 121) !important;
}

/* Form field label */
.mat-form-field-label {
  color: rgb(0, 110, 121) !important;
}













/* Checkbox tick and border color */
::ng-deep .mat-checkbox-checked.mat-accent .mat-checkbox-background {
  background-color: rgb(0, 110, 121) !important;
}

::ng-deep .mat-checkbox.mat-accent .mat-checkbox-frame {
  border-color: rgb(0, 110, 121) !important;
}

/* Outline border of mat-form-field */
::ng-deep .mat-form-field-appearance-outline .mat-form-field-outline {
  color: rgb(0, 110, 121) !important;
  border-color: rgb(0, 110, 121) !important;
}

/* Label color of mat-form-field */
::ng-deep .mat-form-field-label {
  color: rgb(0, 110, 121) !important;
}












/* Apply teal color to checked box background */
.mat-checkbox-checked.mat-accent .mat-checkbox-background {
  background-color: rgb(0, 110, 121) !important;
}

/* Apply teal color to checkbox border */
.mat-checkbox.mat-accent .mat-checkbox-frame {
  border-color: rgb(0, 110, 121) !important;
}



::ng-deep .mat-checkbox-checked.mat-accent .mat-checkbox-background {
  background-color: rgb(0, 110, 121) !important;
}

::ng-deep .mat-checkbox.mat-accent .mat-checkbox-frame {
  border-color: rgb(0, 110, 121) !important;
}












.mat-checkbox-checked.mat-accent .mat-checkbox-background {
  background-color: rgb(0, 110, 121) !important;
}

.mat-checkbox.mat-accent .mat-checkbox-frame {
  border-color: rgb(0, 110, 121) !important;
}

.mat-form-field-appearance-outline .mat-form-field-outline {
  color: rgb(0, 110, 121) !important;
  border-color: rgb(0, 110, 121) !important;
}

.mat-form-field-appearance-outline .mat-form-field-outline-thick {
  color: rgb(0, 110, 121) !important;
  border-color: rgb(0, 110, 121) !important;
}

.mat-form-field-label {
  color: rgb(0, 110, 121) !important;
}









<div class="manager-container">
  <h2>Import-Export Manager</h2>

  <div class="top-bar">
    <!-- Import Tab -->
    <div class="action-box" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon>
      <div>New Import</div>
    </div>

    <!-- Export Tab -->
    <div class="action-box" [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
      <mat-icon>sync_alt</mat-icon>
      <div>Model</div>
    </div>

    <!-- Dropdown visible only when Import tab is active -->
    <div class="dropdown" *ngIf="selectedTab === 'import'">
      <label class="dropdown-label" [class.active]="selectedService" style="color: black;">Choose*</label>

      <mat-form-field appearance="outline" class="dropdown-field" style="color: rgb(0, 110, 121);">
        <mat-label style="color: rgb(0, 110, 121);">Select Recon Service</mat-label>
        <mat-select [(ngModel)]="selectedService" panelClass="custom-select-panel">
          <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
        </mat-select>
      </mat-form-field>
    </div>
  </div>

  <!-- Import Section -->
  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite" style="color: rgb(0, 110, 121);">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button style="background-color: rgb(0, 110, 121); color: white;" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File To Import
      </button>
      <input type="file" #fileInput hidden (change)="onFileSelected($event)" />

      <span *ngIf="!selectedFile">No file chosen</span>
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button style="background-color: rgb(0, 110, 121); color: white;"
            [disabled]="!selectedFile || !selectedService" (click)="upload()">
      Upload
    </button>

    <div *ngIf="showModelTable" class="import-models-table">
      <ag-grid-angular
        class="ag-theme-alpine"
        style="width: 100%; height: 300px;"
        [rowData]="importRowData"
        [columnDefs]="columnDefs"
        rowSelection="multiple"
        (gridReady)="onGridReady($event)">
      </ag-grid-angular>
    </div>
  </div>

  <!-- Export Section -->
  <div *ngIf="selectedTab === 'export'" class="export-section">
    <div style="margin-bottom: 10px;">
      <strong>Export Models ({{ selectedRows.length }} selected)</strong>
    </div>

    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="exportRowData"
      [columnDefs]="exportColumnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)">
    </ag-grid-angular>

    <button mat-raised-button style="margin-top: 10px; background-color: rgb(0, 110, 121); color: white;"
            (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
      Export
    </button>
  </div>

  <!-- Preview Modal -->
  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>



















<div class="manager-container">
  <h2>Import-Export Manager</h2>
  <div class="top-bar">
    <div class="action-box" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon>
      <div>New Import</div>
    </div>
    <div class="action-box" [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
      <mat-icon>sync_alt</mat-icon>
      <div>Model</div>
    </div>
    <div class="dropdown-box">
      <label [class.active]="selectedService">Choose*</label>
      <mat-form-field appearance="outline" class="dropdown">
        <mat-label>Select Recon Service</mat-label>
        <mat-select [(ngModel)]="selectedService">
          <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
        </mat-select>
      </mat-form-field>
    </div>
  </div>

  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite" color="primary">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" #fileInput hidden (change)="onFileSelected($event)" />
      <span>{{ selectedFile?.name || 'No file chosen' }}</span>
    </div>

    <button mat-raised-button color="accent" [disabled]="!selectedFile || !selectedService" (click)="upload()">
      Upload
    </button>
  </div>

  <div *ngIf="selectedTab === 'export'" class="export-section">
    <p><strong>Export Models ({{ selectedRows.length }} selected)</strong></p>
    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="exportData"
      [columnDefs]="exportColumnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" style="margin-top: 10px;" (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
      Export
    </button>
  </div>
</div>


.manager-container {
  font-family: Arial, sans-serif;
  padding: 20px;
}

.top-bar {
  display: flex;
  gap: 20px;
  align-items: center;
  margin-bottom: 20px;
}

.action-box {
  border: 2px solid #007bff;
  border-radius: 8px;
  padding: 12px 16px;
  text-align: center;
  cursor: pointer;
  min-width: 100px;
  color: #007bff;
  background-color: white;
  display: flex;
  flex-direction: column;
  align-items: center;
  transition: all 0.3s ease;
}

.action-box:hover {
  background-color: #007bff;
  color: white;
}

.action-box.active {
  background-color: #007bff;
  color: white;
}

.dropdown-box label {
  font-weight: bold;
  color: #000;
  margin-bottom: 5px;
  display: block;
}

.dropdown-box label.active {
  color: #007bff;
}

.dropdown {
  min-width: 200px;
}

.file-upload {
  display: flex;
  align-items: center;
  gap: 12px;
  margin: 15px 0;
}

mat-checkbox {
  margin-top: 10px;
}

.import-section, .export-section {
  margin-top: 20px;
}






















<div class="manager-container">
  <h2>Import-Export Manager</h2>  <div class="top-bar">
    <div class="action-box" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon>
      <div>New Import</div>
    </div><div class="action-box" [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
  <mat-icon>sync_alt</mat-icon>
  <div>Model</div>
</div>

<div class="dropdown">
  <label class="dropdown-label" [class.active]="selectedService">Choose*</label>
  <mat-form-field appearance="outline" class="dropdown-field">
    <mat-label>Select Recon Service</mat-label>
    <mat-select [(ngModel)]="selectedService">
      <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
    </mat-select>
  </mat-form-field>
</div>

  </div>  <!-- Import Section -->  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite" color="primary">Overwrite</mat-checkbox><div class="file-upload">
  <button mat-raised-button color="primary" (click)="fileInput.click()">
    <mat-icon>attach_file</mat-icon> Select File
  </button>
  <input type="file" #fileInput hidden (change)="onFileSelected($event)" />
  <span *ngIf="!selectedFile">No file chosen</span>
  <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
</div>

<button mat-raised-button color="accent" [disabled]="!selectedFile || !selectedService" (click)="upload()">
  Upload
</button>

<div *ngIf="showModelTable" class="import-models-table">
  <ag-grid-angular
    class="ag-theme-alpine"
    style="width: 100%; height: 300px;"
    [rowData]="importData"
    [columnDefs]="columnDefs"
    rowSelection="multiple"
    (gridReady)="onGridReady($event)">
  </ag-grid-angular>
</div>

  </div>  <!-- Export Section -->  <div *ngIf="selectedTab === 'export'" class="export-section">
    <div style="margin-bottom: 10px;">
      <strong>Export Models ({{ selectedRows.length }} selected)</strong>
    </div>
    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="exportData"
      [columnDefs]="exportColumnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)">
    </ag-grid-angular><button mat-raised-button color="primary" style="margin-top: 10px;" (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
  Export
</button>

  </div>  <!-- Preview Modal -->  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>




.manager-container {
  font-family: Arial, sans-serif;
  padding: 20px;
}

.top-bar {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-bottom: 20px;
}

mat-form-field.dropdown {
  min-width: 200px;
  margin: 0;
}

button.mat-raised-button {
  border-radius: 6px;
  text-transform: none;
}

button.mat-raised-button.active {
  background-color: #007bff;
  color: white;
}

button.mat-raised-button:hover {
  background-color: #007bff !important;
  color: white !important;
}

.import-section,
.export-section {
  margin-top: 20px;
  margin-bottom: 20px;
}

.file-upload {
  margin-top: 15px;
  display: flex;
  align-items: center;
  gap: 10px;
}

span {
  margin-left: 10px;
  font-style: italic;
}

.import-models-table,
.export-models-table {
  margin-top: 20px;
}

mat-checkbox.mat-checkbox-checked .mat-checkbox-background,
mat-checkbox.mat-accent .mat-checkbox-background {
  background-color: #007bff !important;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.4);
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

h2 {
  color: #1a1a1a;
  margin-bottom: 10px;
}

/* Highlight headings on tab selection */
.top-bar button.active h2,
.top-bar button:hover h2,
mat-label {
  color: #007bff;
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
      console.log('Upload successful');
      this.fetchImportTable();  // ✅ Ensure table loads after upload
      this.selectedFile = null;
      this.showModelTable = true;  // ✅ This must be true to display the AG Grid
    },
    error: (err) => {
      console.error('Upload failed:', err);
      alert('Upload failed.');
    }
  });
}


fetchImportTable() {
  this.service.getImportMetadata().subscribe({
    next: (data) => {
      console.log('Fetched import metadata:', data);
      this.rowData = data;
    },
    error: (err) => {
      console.error('Failed to load import metadata:', err);
    }
  });
}















getImportMetadata(): Observable<any[]> {
  return this.http.get<any[]>('http://localhost:8080/api/import/metadata');
}










<div class="manager-container">
  <h2>Import-Export Manager</h2>  <div class="top-bar">
    <button mat-raised-button color="primary" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon> New Import
    </button><button mat-raised-button color="warn" [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
  <mat-icon>sync_alt</mat-icon> Model
</button>

<mat-form-field appearance="outline" class="dropdown">
  <mat-label>Select Recon Service</mat-label>
  <mat-select [(ngModel)]="selectedService">
    <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
  </mat-select>
</mat-form-field>

  </div>  <!-- Import Section -->  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox><div class="file-upload">
  <button mat-raised-button color="primary" (click)="fileInput.click()">
    <mat-icon>attach_file</mat-icon> Select File
  </button>
  <input type="file" #fileInput hidden (change)="onFileSelected($event)" />
  <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
</div>

<button mat-raised-button color="accent" [disabled]="!selectedFile || !selectedService" (click)="upload()">
  Upload
</button>

<div *ngIf="showModelTable" class="import-models-table">
  <ag-grid-angular
    class="ag-theme-alpine"
    style="width: 100%; height: 300px;"
    [rowData]="importData"
    [columnDefs]="columnDefs"
    rowSelection="multiple"
    (gridReady)="onGridReady($event)">
  </ag-grid-angular>
</div>

  </div>  <!-- Export Section -->  <div *ngIf="selectedTab === 'export'" class="export-section">
    <div style="margin-bottom: 10px;">
      <strong>Export Models ({{ selectedRows.length }} selected)</strong>
    </div><ag-grid-angular
  class="ag-theme-alpine"
  style="width: 100%; height: 300px;"
  [rowData]="exportData"
  [columnDefs]="exportColumnDefs"
  rowSelection="multiple"
  (selectionChanged)="onSelectionChanged($event)">
</ag-grid-angular>

<button mat-raised-button color="primary" style="margin-top: 10px;" (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
  Export
</button>

  </div>  <!-- Preview Modal -->  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>




ts
import { Component, OnInit, ViewChild } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
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

  importRowData: any[] = [];
  exportRowData: any[] = [];
  selectedRows: any[] = [];

  showModal = false;
  previewJson: string | null = null;
  showImportTable = false;

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

  exportColumnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'name', headerName: 'Model' },
    { field: 'model_mode', headerName: 'Mode' },
    { field: 'frequency', headerName: 'Frequency' },
    { field: 'context', headerName: 'Context' },
    { field: 'service', headerName: 'Service' }
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
      next: (data) => this.services = data,
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
        this.selectedFile = null;
        this.fetchImportModels();
        this.showImportTable = true;
      },
      error: () => alert('Upload failed.')
    });
  }

  fetchImportModels() {
    this.service.getImportModels().subscribe({
      next: (data) => this.importRowData = data,
      error: () => alert('Failed to load import models.')
    });
  }

  fetchExportModels() {
    this.service.getExportModels().subscribe({
      next: (data) => this.exportRowData = data,
      error: () => alert('Failed to load export models.')
    });
  }

  onSelectionChanged(event: any) {
    this.selectedRows = event.api.getSelectedRows();
  }

  exportSelectedModels() {
    const modelNames = this.selectedRows.map(row => row.name);
    if (modelNames.length === 0) {
      alert('No models selected for export.');
      return;
    }

    this.service.downloadModels(modelNames).subscribe(blob => {
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'models.zip';
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      window.URL.revokeObjectURL(url);
    });
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

  get rowData() {
    return this.selectedTab === 'import' ? this.importRowData : this.exportRowData;
  }
}



import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ImportExportService {
  private baseImportUrl = 'http://localhost:8080/api/import';
  private baseExportUrl = 'http://localhost:8080/api/export';

  constructor(private http: HttpClient) {}

  getServices(): Observable<string[]> {
    return this.http.get<string[]>(`${this.baseImportUrl}/services`);
  }

  upload(formData: FormData): Observable<any> {
    return this.http.post(`${this.baseImportUrl}/upload`, formData);
  }

  getImportModels(): Observable<any[]> {
    return this.http.get<any[]>(`${this.baseImportUrl}/metadata`);
  }

  getExportModels(): Observable<any[]> {
    return this.http.get<any[]>(`${this.baseExportUrl}/export-models`);
  }

  getJsonPreview(filename: string): Observable<string> {
    return this.http.get(`${this.baseImportUrl}/json/${filename}`, { responseType: 'text' });
  }

  downloadModels(modelNames: string[]): Observable<Blob> {
    return this.http.post(`${this.baseExportUrl}/download`, modelNames, { responseType: 'blob' });
  }
}


@GetMapping("/metadata")
public ResponseEntity<List<ImportMetadata>> getAllImportMetadata() {
    List<ImportMetadata> list = repository.findAll(Sort.by(Sort.Direction.DESC, "uploadedAt"));
    return ResponseEntity.ok(list);
}

model export controller 
@PostMapping("/download")
public ResponseEntity<Resource> downloadModels(@RequestBody List<String> modelNames) throws IOException {
    List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);

    if (models.isEmpty()) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(null);
    }

    ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
    ZipOutputStream zos = new ZipOutputStream(zipOutStream);

    for (ExportModelDTO model : models) {
        String safeName = model.getName().replaceAll("[^a-zA-Z0-9_-]", "_");
        String json = new ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(model);

        ZipEntry entry = new ZipEntry(safeName + ".json");
        zos.putNextEntry(entry);
        zos.write(json.getBytes(StandardCharsets.UTF_8));
        zos.closeEntry();
    }

    zos.finish();
    zos.close();

    ByteArrayResource resource = new ByteArrayResource(zipOutStream.toByteArray());

    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(resource);
}







package com.example.work.model;

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















<div class="manager-container">
  <h2>Import-Export Manager</h2>

  <!-- Top Bar -->
  <div class="top-bar">
    <button mat-raised-button color="primary" [class.active]="selectedTab === 'import'" (click)="selectTab('import')">
      <mat-icon>cloud_upload</mat-icon> New Import
    </button>

    <button mat-raised-button color="accent" [class.active]="selectedTab === 'export'" (click)="selectTab('export')">
      <mat-icon>sync_alt</mat-icon> Model
    </button>

    <mat-form-field appearance="outline" class="dropdown">
      <mat-label>Select Recon Service</mat-label>
      <mat-select [(ngModel)]="selectedService">
        <mat-option *ngFor="let s of services" [value]="s">{{ s }}</mat-option>
      </mat-select>
    </mat-form-field>
  </div>

  <!-- Import Section -->
  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" #fileInput hidden (change)="onFileSelected($event)" />
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button color="accent" (click)="upload()" [disabled]="!selectedFile || !selectedService">
      Upload
    </button>

    <div *ngIf="rowData.length > 0" class="import-models-table">
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

  <!-- Export Section -->
  <div *ngIf="selectedTab === 'export'" class="export-section">
    <div style="margin-bottom: 10px;">
      <strong>Export Models ({{ selectedRows.length }} selected)</strong>
    </div>

    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="rowData"
      [columnDefs]="exportColumnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" style="margin-top: 10px;" (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
      Export
    </button>
  </div>

  <!-- Preview Modal -->
  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>








// Updated FileImportController.java @RestController @RequestMapping("/api/import") @CrossOrigin public class FileImportController {

@Autowired
private ImportMetadataRepository repository;

@Autowired
private ReconServiceRepository reconServiceRepository;

private final String uploadDir = System.getProperty("user.home") + "/Downloads/";

@PostMapping("/upload")
public ResponseEntity<Map<String, String>> upload(@RequestParam("file") MultipartFile file,
                                                  @RequestParam("service") String service,
                                                  @RequestParam("overwrite") boolean overwrite) {
    try {
        Path targetPath = Paths.get(uploadDir + file.getOriginalFilename());
        Files.createDirectories(targetPath.getParent());
        Files.write(targetPath, file.getBytes(), StandardOpenOption.CREATE);

        ImportMetadata metadata = new ImportMetadata();
        metadata.setFilename(file.getOriginalFilename());
        metadata.setService(service);
        metadata.setOverwriteFlag(overwrite ? 'Y' : 'N');
        repository.save(metadata);

        Map<String, String> response = new HashMap<>();
        response.put("message", "File uploaded and metadata saved.");
        return ResponseEntity.ok(response);
    } catch (IOException e) {
        Map<String, String> error = new HashMap<>();
        e.printStackTrace();
        error.put("error", "Upload failed: " + e.getMessage());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
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

@GetMapping("/services")
public ResponseEntity<List<String>> getAvailableServices() {
    List<String> services = reconServiceRepository.findAllServiceNames();
    return ResponseEntity.ok(services);
}

}

// Updated ModelExportController.java @RestController @RequestMapping("/api/export") public class ModelExportController {

@Autowired
private ModelExportService exportService;

@GetMapping("/export-models")
public List<ExportModelDTO> getAllExportModels() {
    String sql = "SELECT name, description, service, context, frequency, model_mode FROM recon_models";
    return exportService.getAllModels(sql);
}

@PostMapping("/download")
public ResponseEntity<Resource> downloadModels(@RequestBody List<String> modelNames) throws IOException {
    List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);
    if (models.isEmpty()) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(null);
    }

    ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
    ZipOutputStream zos = new ZipOutputStream(zipOutStream);

    for (ExportModelDTO model : models) {
        String safeName = model.getName().replaceAll("[^a-zA-Z0-9_-]", "_");
        String json = new ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(model);
        ZipEntry entry = new ZipEntry(safeName + ".json");
        zos.putNextEntry(entry);
        zos.write(json.getBytes(StandardCharsets.UTF_8));
        zos.closeEntry();
    }

    zos.finish();
    zos.close();

    ByteArrayResource resource = new ByteArrayResource(zipOutStream.toByteArray());
    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(resource);
}

}

// DTO and Model remain unchanged

// Ensure the method in repository is correct public interface ReconServiceRepository { @Query("SELECT DISTINCT service FROM recon_models") List<String> findAllServiceNames(); }




















import { Component, OnInit } from '@angular/core';
import { ImportExportService } from './import-export.service';
import { ColDef } from 'ag-grid-community';

@Component({
  selector: 'app-import-export-manager',
  standalone: true,
  templateUrl: './import-export-manager.component.html',
  styleUrls: ['./import-export-manager.component.css'],
  imports: [
    // Your Angular Material + AG Grid modules
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

  exportColumnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'name', headerName: 'Model' },
    { field: 'model_mode', headerName: 'Mode' },
    { field: 'frequency', headerName: 'Frequency' },
    { field: 'context', headerName: 'Context' },
    { field: 'service', headerName: 'Service' }
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
        this.fetchImportModels();
        this.selectedFile = null;
      },
      error: () => alert('Upload failed.')
    });
  }

  fetchImportModels() {
    this.service.getExportModels().subscribe({
      next: (data) => (this.rowData = data),
      error: () => alert('Failed to load import data.')
    });
  }

  fetchExportModels() {
    this.service.getExportModels().subscribe({
      next: (data) => (this.rowData = data),
      error: () => alert('Failed to load export models.')
    });
  }

  onSelectionChanged(event: any) {
    this.selectedRows = event.api.getSelectedRows();
  }

  exportSelectedModels() {
    const modelNames = this.selectedRows.map(row => row.name);
    if (modelNames.length === 0) {
      alert('Please select at least one model to export.');
      return;
    }

    this.service.downloadModels(modelNames).subscribe(blob => {
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'models.zip';
      document.body.appendChild(a);
      a.click();
      a.remove();
      window.URL.revokeObjectURL(url);
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


























ts
// Updated import-export-manager.component.ts import { Component, OnInit } from '@angular/core'; import { HttpClient } from '@angular/common/http'; import { ColDef } from 'ag-grid-community'; import { ImportExportService } from './import-export.service';

@Component({ selector: 'app-import-export-manager', standalone: true, templateUrl: './import-export-manager.component.html', styleUrls: ['./import-export-manager.component.css'] }) export class ImportExportManagerComponent implements OnInit { selectedTab: 'import' | 'export' = 'import'; services: string[] = []; selectedService = ''; overwrite = false; selectedFile: File | null = null; rowData: any[] = []; selectedRows: any[] = []; showModal = false; previewJson: string | null = null; showModelTable = false;

columnDefs: ColDef[] = [ { headerName: '', checkboxSelection: true, width: 50 }, { field: 'filename', headerName: 'Model' }, { field: 'overwriteFlag', headerName: 'Mode' }, { field: 'uploadedAt', headerName: 'Uploaded At' }, { field: 'service', headerName: 'Service' }, { headerName: 'Actions', cellRenderer: () => '<button class="preview-btn">Preview</button>', width: 100 } ];

exportColumnDefs: ColDef[] = [ { headerName: '', checkboxSelection: true, width: 50 }, { field: 'name', headerName: 'Model' }, { field: 'model_mode', headerName: 'Mode' }, { field: 'frequency', headerName: 'Frequency' }, { field: 'context', headerName: 'Context' }, { field: 'service', headerName: 'Service' } ];

constructor(private service: ImportExportService) {}

ngOnInit(): void { this.fetchServices(); this.fetchExportModels(); }

selectTab(tab: 'import' | 'export') { this.selectedTab = tab; if (tab === 'export') this.fetchExportModels(); }

fetchServices() { this.service.getServices().subscribe({ next: (data) => (this.services = data), error: () => alert('Failed to load services.') }); }

onFileSelected(event: any) { this.selectedFile = event.target.files[0]; }

upload() { if (!this.selectedFile || !this.selectedService) { alert('Please select a service and a file.'); return; } const formData = new FormData(); formData.append('file', this.selectedFile); formData.append('service', this.selectedService); formData.append('overwrite', String(this.overwrite));

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

fetchExportModels() { this.service.getExportModels().subscribe((data) => (this.rowData = data)); }

onSelectionChanged(event: any) { this.selectedRows = event.api.getSelectedRows(); }

exportSelectedModels() { const modelNames = this.selectedRows.map((row) => row.name); this.service.downloadModels(modelNames).subscribe((blob) => { const url = window.URL.createObjectURL(blob); const a = document.createElement('a'); a.href = url; a.download = 'models.zip'; document.body.appendChild(a); a.click(); a.remove(); window.URL.revokeObjectURL(url); }); }

onGridReady(params: any) { params.api.addEventListener('cellClicked', (event: any) => { if ( event.colDef.headerName === 'Actions' && event.event.target.classList.contains('preview-btn') ) { this.loadJsonPreview(event.data.filename); } }); }

loadJsonPreview(filename: string) { this.service.getJsonPreview(filename).subscribe({ next: (json) => { this.previewJson = json; this.showModal = true; }, error: () => alert('Failed to load JSON preview.') }); }

closeModal() { this.showModal = false; this.previewJson = null; } }

HTML
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

  <!-- Import Section -->
  <div *ngIf="selectedTab === 'import'" class="import-section">
    <mat-checkbox [(ngModel)]="overwrite">Overwrite</mat-checkbox>

    <div class="file-upload">
      <button mat-raised-button color="primary" (click)="fileInput.click()">
        <mat-icon>attach_file</mat-icon> Select File
      </button>
      <input type="file" #fileInput hidden (change)="onFileSelected($event)" />
      <span *ngIf="selectedFile">{{ selectedFile.name }}</span>
    </div>

    <button mat-raised-button color="accent" [disabled]="!selectedFile || !selectedService" (click)="upload()">
      Upload
    </button>

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

  <!-- Export Section -->
  <div *ngIf="selectedTab === 'export'" class="export-section">
    <div style="margin-bottom: 10px;">
      <strong>Export Models ({{ selectedRows.length }} selected)</strong>
    </div>

    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="rowData"
      [columnDefs]="exportColumnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" style="margin-top: 10px;" (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
      Export
    </button>
  </div>

  <!-- Preview Modal -->
  <div *ngIf="showModal" class="modal">
    <div class="modal-content">
      <h3>JSON Preview</h3>
      <pre>{{ previewJson }}</pre>
      <button mat-button (click)="closeModal()">Close</button>
    </div>
  </div>
</div>


 


----------------------------------
import
package com.yourcompany.importexport.controller;

import com.yourcompany.importexport.model.ImportMetadata;
import com.yourcompany.importexport.repository.ImportMetadataRepository;
import com.yourcompany.importexport.repository.ReconServiceRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.ByteArrayResource;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.util.*;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

@RestController
@RequestMapping("/api/import")
@CrossOrigin
public class FileImportController {

    private final String uploadDir = System.getProperty("user.home") + "/Downloads/";

    @Autowired
    private ImportMetadataRepository repository;

    @Autowired
    private ReconServiceRepository reconServiceRepository;

    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam("service") String service,
            @RequestParam("overwrite") boolean overwrite) {

        try {
            Path targetPath = Paths.get(uploadDir + file.getOriginalFilename());
            Files.createDirectories(targetPath.getParent());
            Files.write(targetPath, file.getBytes(), StandardOpenOption.CREATE);

            ImportMetadata metadata = new ImportMetadata();
            metadata.setFilename(file.getOriginalFilename());
            metadata.setService(service);
            metadata.setOverwriteFlag(overwrite ? "Y" : "N");

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

    @GetMapping("/json/{filename}")
    public ResponseEntity<String> getUploadedJson(@PathVariable String filename) {
        try {
            Path path = Paths.get(uploadDir + filename);
            if (!Files.exists(path)) {
                return ResponseEntity.status(HttpStatus.NOT_FOUND).body("File not found.");
            }
            String content = Files.readString(path, StandardCharsets.UTF_8);
            return ResponseEntity.ok(content);
        } catch (IOException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("Failed to read file: " + e.getMessage());
        }
    }

    @GetMapping("/services")
    public ResponseEntity<List<String>> getAvailableServices() {
        List<String> services = reconServiceRepository.findAllServiceNames();
        return ResponseEntity.ok(services);
    }

    @GetMapping("/export-models")
    public ResponseEntity<List<ImportMetadata>> getAllUploadedModels() {
        List<ImportMetadata> all = repository.findAll();
        return ResponseEntity.ok(all);
    }
}


export 

package com.yourcompany.importexport.controller;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.io.ByteArrayOutputStream;
import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

@RestController
@RequestMapping("/api/export")
@CrossOrigin
public class ExportController {

    private final String uploadDir = System.getProperty("user.home") + "/Downloads/";

    @PostMapping("/download")
    public ResponseEntity<byte[]> downloadModels(@RequestBody List<String> modelNames) {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             ZipOutputStream zos = new ZipOutputStream(baos)) {

            for (String name : modelNames) {
                Path path = Paths.get(uploadDir + name);
                if (Files.exists(path)) {
                    String content = Files.readString(path, StandardCharsets.UTF_8);
                    ZipEntry entry = new ZipEntry(name);
                    zos.putNextEntry(entry);
                    zos.write(content.getBytes(StandardCharsets.UTF_8));
                    zos.closeEntry();
                }
            }

            zos.finish();
            byte[] zipBytes = baos.toByteArray();

            return ResponseEntity.ok()
                    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip")
                    .contentType(MediaType.APPLICATION_OCTET_STREAM)
                    .body(zipBytes);

        } catch (IOException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(("Failed to generate ZIP: " + e.getMessage()).getBytes());
        }
    }
}










@GetMapping("/import/export-models")
public ResponseEntity<List<ExportModelDTO>> getExportModels() {
    return ResponseEntity.ok(exportService.getAllExportModels());
}


@GetMapping("/import/preview")
public ResponseEntity<String> previewJson(@RequestParam String filename) {
    return ResponseEntity.ok(previewService.loadJson(filename));
}







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

  exportColumnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'name', headerName: 'Model' },
    { field: 'mode', headerName: 'Mode' },
    { field: 'frequency', headerName: 'Frequency' },
    { field: 'context', headerName: 'Context' },
    { field: 'service', headerName: 'Service' }
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
      next: (data) => this.services = data,
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
    this.service.getExportModels().subscribe(data => this.rowData = data);
  }

  onSelectionChanged(event: any) {
    this.selectedRows = event.api.getSelectedRows();
  }

  exportSelectedModels() {
    const modelNames = this.selectedRows.map(row => row.name);
    this.service.downloadModels(modelNames).subscribe(blob => {
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'models.zip';
      document.body.appendChild(a);
      a.click();
      a.remove();
      window.URL.revokeObjectURL(url);
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

  <!-- Import Section -->
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

  <!-- Export Section -->
  <div *ngIf="selectedTab === 'export'" class="export-section">
    <div style="margin-bottom: 10px;">
      <strong>Export Models ({{ selectedRows.length }}) selected</strong>
    </div>

    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 300px;"
      [rowData]="rowData"
      [columnDefs]="exportColumnDefs"
      rowSelection="multiple"
      (selectionChanged)="onSelectionChanged($event)">
    </ag-grid-angular>

    <button mat-raised-button color="primary" style="margin-top: 10px;" (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
      Export
    </button>
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
















<div style="padding: 20px;">
  <h2>Import-Export Manager</h2>

  <div style="display: flex; gap: 40px;">
    <!-- Import Section -->
    <div style="text-align: center;">
      <p><strong>Import</strong></p>
      <button mat-raised-button color="primary" style="min-width: 120px;">
        <mat-icon>cloud_upload</mat-icon>
        New Import
      </button>
    </div>

    <!-- Export Section -->
    <div style="text-align: center;">
      <p><strong>Export</strong></p>
      <button mat-raised-button color="accent" style="min-width: 120px;">
        <mat-icon>compare_arrows</mat-icon>
        Model
      </button>
    </div>
  </div>
</div>
















@PostMapping("/download")
public ResponseEntity<Resource> downloadModels(@RequestBody List<String> modelNames) throws IOException {
    List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);

    if (models.isEmpty()) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(null);
    }

    ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
    ZipOutputStream zos = new ZipOutputStream(zipOutStream);

    for (ExportModelDTO model : models) {
        String safeName = model.getName().replaceAll("[^a-zA-Z0-9_-]", "_"); // prevent invalid filenames
        String json = new ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(model);

        ZipEntry entry = new ZipEntry(safeName + ".json");
        zos.putNextEntry(entry);
        zos.write(json.getBytes(StandardCharsets.UTF_8));
        zos.closeEntry();
    }

    zos.finish(); // finalize properly
    zos.close();

    ByteArrayResource resource = new ByteArrayResource(zipOutStream.toByteArray());

    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(resource);
}















@PostMapping("/api/export/download")
public ResponseEntity<Resource> downloadModels(@RequestBody List<String> modelNames) throws IOException {
    ByteArrayOutputStream zipOut = new ByteArrayOutputStream();
    ZipOutputStream zos = new ZipOutputStream(zipOut);

    for (String modelName : modelNames) {
        Optional<ModelEntity> model = modelRepository.findByName(modelName);
        if (model.isPresent()) {
            String json = new ObjectMapper().writeValueAsString(model.get());
            ZipEntry entry = new ZipEntry(modelName + ".json");
            zos.putNextEntry(entry);
            zos.write(json.getBytes());
            zos.closeEntry();
        }
    }

    zos.close();
    ByteArrayResource resource = new ByteArrayResource(zipOut.toByteArray());

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(resource);
}


















@PostMapping("/download")
public ResponseEntity<?> downloadModels(@RequestBody List<String> modelNames) throws IOException {
    List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);

    ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
    ZipOutputStream zos = new ZipOutputStream(zipOutStream);

    for (ExportModelDTO model : models) {
        String json = new ObjectMapper()
                .writerWithDefaultPrettyPrinter()
                .writeValueAsString(model);

        ZipEntry entry = new ZipEntry(model.getName() + ".json");
        zos.putNextEntry(entry);
        zos.write(json.getBytes(StandardCharsets.UTF_8));
        zos.closeEntry();
    }

    zos.finish(); // ✅ Important to finalize the zip structure
    zos.close();

    byte[] zipBytes = zipOutStream.toByteArray();

    HttpHeaders headers = new HttpHeaders();
    headers.set(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip");

    return ResponseEntity.ok()
            .headers(headers)
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(zipBytes);
}
















@PostMapping("/download")
public ResponseEntity<?> downloadModels(@RequestBody List<String> modelNames) throws IOException {
    List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);

    ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
    ZipOutputStream zos = new ZipOutputStream(zipOutStream);

    for (ExportModelDTO model : models) {
        String json = new ObjectMapper()
                .writerWithDefaultPrettyPrinter()
                .writeValueAsString(model);

        ZipEntry entry = new ZipEntry(model.getName() + ".json");
        zos.putNextEntry(entry);
        zos.write(json.getBytes(StandardCharsets.UTF_8));
        zos.closeEntry(); // ✅ Correctly close the entry after writing
    }

    zos.close(); // ✅ Close the zip stream after the loop

    byte[] zipBytes = zipOutStream.toByteArray();

    HttpHeaders headers = new HttpHeaders();
    headers.set(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip");

    return ResponseEntity.ok()
            .headers(headers)
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(zipBytes);
}







@RestController
@RequestMapping("/api/export")
public class ModelExportController {

    @Autowired
    private ModelExportService exportService;

    @GetMapping("/export-models")
    public List<ExportModelDTO> getAllExportModels() {
        String sql = "SELECT name, description, service, context, frequency, model_mode AS mode FROM recon_models";
        return exportService.getAllModels(sql);  // This must now compile!
    }

    @PostMapping("/download")
    public ResponseEntity<?> downloadModels(@RequestBody List<String> modelNames) throws IOException {
        List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);
        ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
        ZipOutputStream zos = new ZipOutputStream(zipOutStream);

        for (ExportModelDTO model : models) {
            String json = new ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(model);
            ZipEntry entry = new ZipEntry(model.getName() + ".json");
            zos.putNextEntry(entry);
            zos.write(json.getBytes(StandardCharsets.UTF_8));
            zos.closeEntry();
        }

        zos.close();

        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip");

        return ResponseEntity.ok()
                .headers(headers)
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .body(zipOutStream.toByteArray());
    }
}








@Service
public class ModelExportService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public List<ExportModelDTO> getModelsByNames(List<String> modelNames) {
        String inSql = String.join(",", Collections.nCopies(modelNames.size(), "?"));
        String sql = "SELECT name, description, service, context, frequency, model_mode AS mode FROM recon_models WHERE name IN (" + inSql + ")";

        return jdbcTemplate.query(sql, modelNames.toArray(), (rs, rowNum) -> {
            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(rs.getString("name"));
            dto.setDescription(rs.getString("description"));
            dto.setService(rs.getString("service"));
            dto.setContext(rs.getString("context"));
            dto.setFrequency(rs.getString("frequency"));
            dto.setMode(rs.getString("mode")); // Note: mapped from model_mode AS mode
            return dto;
        });
    }

    // ✅ This is the missing method
    public List<ExportModelDTO> getAllModels(String sql) {
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(rs.getString("name"));
            dto.setDescription(rs.getString("description"));
            dto.setService(rs.getString("service"));
            dto.setContext(rs.getString("context"));
            dto.setFrequency(rs.getString("frequency"));
            dto.setMode(rs.getString("mode")); // From alias in SQL: model_mode AS mode
            return dto;
        });
    }
}










package your.package.name;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import java.util.Collections;
import java.util.List;
import com.your.package.ExportModelDTO;

@Service
public class ModelExportService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public List<ExportModelDTO> getModelsByNames(List<String> modelNames) {
        String inSql = String.join(",", Collections.nCopies(modelNames.size(), "?"));
        String sql = "SELECT name, description, service, context, frequency, model_mode AS mode FROM recon_models WHERE name IN (" + inSql + ")";

        return jdbcTemplate.query(sql, modelNames.toArray(), (rs, rowNum) -> {
            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(rs.getString("name"));
            dto.setDescription(rs.getString("description"));
            dto.setService(rs.getString("service"));
            dto.setContext(rs.getString("context"));
            dto.setFrequency(rs.getString("frequency"));
            dto.setMode(rs.getString("mode"));  // AS mode is mapped here
            return dto;
        });
    }

    public List<ExportModelDTO> getAllModels(String sql) {
        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(rs.getString("name"));
            dto.setDescription(rs.getString("description"));
            dto.setService(rs.getString("service"));
            dto.setContext(rs.getString("context"));
            dto.setFrequency(rs.getString("frequency"));
            dto.setMode(rs.getString("mode"));  // again, using alias 'mode'
            return dto;
        });
    }
}
















<div *ngIf="selectedTab === 'export'" class="export-section">
  <div style="margin-bottom: 10px;">
    <strong>Export Models ({{ selectedRows.length }} selected)</strong>
  </div>

  <ag-grid-angular
    class="ag-theme-alpine"
    style="width: 100%; height: 300px;"
    [rowData]="rowData"
    [columnDefs]="exportColumnDefs"
    rowSelection="multiple"
    (selectionChanged)="onSelectionChanged($event)">
  </ag-grid-angular>

  <button mat-raised-button color="primary" style="margin-top: 10px;" (click)="exportSelectedModels()" [disabled]="selectedRows.length === 0">
    Export
  </button>
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

  exportColumnDefs: ColDef[] = [
    { headerName: '', checkboxSelection: true, width: 50 },
    { field: 'name', headerName: 'Model' },
    { field: 'mode', headerName: 'Mode' },
    { field: 'frequency', headerName: 'Frequency' },
    { field: 'context', headerName: 'Context' },
    { field: 'service', headerName: 'Service' }
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

  onSelectionChanged(event: any) {
    this.selectedRows = event.api.getSelectedRows();
  }

  exportSelectedModels() {
    const modelNames = this.selectedRows.map(row => row.name);
    this.service.downloadModels(modelNames).subscribe(blob => {
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'models.zip';
      document.body.appendChild(a);
      a.click();
      a.remove();
      window.URL.revokeObjectURL(url);
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



downloadModels(modelNames: string[]) {
  return this.http.post(
    '/api/export/download',
    modelNames,
    { responseType: 'blob' }
  );
}

getExportModels() {
  return this.http.get<any[]>('/api/export/models'); // your GET endpoint to load model list
}




@GetMapping("/export-models")
public List<ExportModelDTO> getAllExportModels() {
    String sql = "SELECT name, description, service, context, frequency, model_mode AS mode FROM recon_models";

    return exportService.getAllModels(sql);
}















import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class ExportModelDTO {
    private String name;
    private String description;
    private String service;
    private String context;
    private String frequency;
    private String mode;
}

model export controller 
@RestController
@RequestMapping("/api/export")
public class ModelExportController {

    @Autowired
    private ModelExportService exportService;

    @PostMapping("/download")
    public ResponseEntity<?> downloadModels(@RequestBody List<String> modelNames) throws IOException {
        List<ExportModelDTO> models = exportService.getModelsByNames(modelNames);

        ByteArrayOutputStream zipOutStream = new ByteArrayOutputStream();
        ZipOutputStream zos = new ZipOutputStream(zipOutStream);

        for (ExportModelDTO model : models) {
            String json = new ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(model);
            ZipEntry entry = new ZipEntry(model.getName() + ".json");
            zos.putNextEntry(entry);
            zos.write(json.getBytes(StandardCharsets.UTF_8));
            zos.closeEntry();
        }

        zos.close();

        byte[] zipBytes = zipOutStream.toByteArray();
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=models.zip");

        return ResponseEntity.ok()
                .headers(headers)
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .body(zipBytes);
    }
}

model export service
@Service
public class ModelExportService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public List<ExportModelDTO> getModelsByNames(List<String> modelNames) {
        String inSql = String.join(",", Collections.nCopies(modelNames.size(), "?"));
        String sql = "SELECT name, description, service, context, frequency, mode FROM recon_models WHERE name IN (" + inSql + ")";

        return jdbcTemplate.query(sql, modelNames.toArray(), (rs, rowNum) -> {
            ExportModelDTO dto = new ExportModelDTO();
            dto.setName(rs.getString("name"));
            dto.setDescription(rs.getString("description"));
            dto.setService(rs.getString("service"));
            dto.setContext(rs.getString("context"));
            dto.setFrequency(rs.getString("frequency"));
            dto.setMode(rs.getString("mode"));
            return dto;
        });
    }
}

CREATE TABLE recon_models (
    id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR2(100),
    description VARCHAR2(500),
    service VARCHAR2(100),
    context VARCHAR2(100),
    frequency VARCHAR2(100),
    mode VARCHAR2(50)
);

INSERT INTO recon_models (name, description, service, context, frequency, mode) VALUES
('Germany_Holdings', 'Model for Germany Holdings', 'IMCIBWM_Recon', 'Securities', 'On Demand', 'Continuous');

INSERT INTO recon_models (name, description, service, context, frequency, mode) VALUES
('Germany_Nostro', 'Model for Germany Nostro', 'IMCIBWM_Recon', 'Cash', 'On Demand', 'Continuous');

INSERT INTO recon_models (name, description, service, context, frequency, mode) VALUES
('Switzerland_Settlement', 'Swiss settlement model', 'IMCIBWM_Recon', 'Fwd Rate Agr', 'On Demand', 'Continuous');






















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







  
