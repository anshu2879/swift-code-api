# swift-code-api

import (
    "log"
    "github.com/YourUsername/swift-code-api/internal/api"
    "github.com/YourUsername/swift-code-api/internal/db"
    "github.com/gin-gonic/gin"
)

func main() {
    repo, err := db.NewRepository("postgres://user:password@db:5432/swift?sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    handler := api.NewHandler(repo)
    r := gin.Default()
    api.SetupRoutes(r, handler)
    if err := r.Run(":8080"); err != nil {
        log.Fatal(err)
    }
}
Replace YourUsername with your GitHub username.
Create a folder called internal in the swift-code-api folder.
Inside internal, create three folders: api, db, and parser.
Create these files with the code below:
File: internal/parser/parser.go
go
package parser

import (
    "encoding/csv"
    "io"
    "strings"
)

type SwiftCode struct {
    SwiftCode    string
    BankName     string
    Address      string
    CountryISO2  string
    CountryName  string
    IsHeadquarter bool
}

func ParseCSV(r io.Reader) ([]SwiftCode, error) {
    reader := csv.NewReader(r)
    var swiftCodes []SwiftCode
    header := true

    for {
        record, err := reader.Read()
        if err == io.EOF {
            break
        }
        if err != nil {
            return nil, err
        }
        if header {
            header = false
            continue
        }

        swiftCode := SwiftCode{
            SwiftCode:    record[0],
            BankName:     record[1],
            Address:      record[2],
            CountryISO2:  strings.ToUpper(record[3]),
            CountryName:  strings.ToUpper(record[4]),
            IsHeadquarter: strings.HasSuffix(record[0], "XXX"),
        }
        swiftCodes = append(swiftCodes, swiftCode)
    }
    return swiftCodes, nil
}
File: internal/db/models.go
go
package db

type SwiftCode struct {
    SwiftCode    string json:"swiftCode"
    BankName     string json:"bankName"
    Address      string json:"address"
    CountryISO2  string json:"countryISO2"
    CountryName  string json:"countryName"
    IsHeadquarter bool   json:"isHeadquarter"
}
File: internal/db/repository.go
go
package db

import (
    "database/sql"
    "strings"
    _ "github.com/lib/pq"
)

type Repository struct {
    db *sql.DB
}

func NewRepository(connStr string) (*Repository, error) {
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }
    return &Repository{db: db}, nil
}

func (r *Repository) SaveSwiftCode(sc SwiftCode) error {
    query := `
        INSERT INTO swift_codes (swift_code, bank_name, address, country_iso2, country_name, is_headquarter)
        VALUES ($1, $2, $3, $4, $5, $6)
        ON CONFLICT (swift_code) DO NOTHING`
    _, err := r.db.Exec(query, sc.SwiftCode, sc.BankName, sc.Address, sc.CountryISO2, sc.CountryName, sc.IsHeadquarter)
    return err
}

func (r *Repository) GetSwiftCode(swiftCode string) (SwiftCode, []SwiftCode, error) {
    var sc SwiftCode
    query := `SELECT swift_code, bank_name, address, country_iso2, country_name, is_headquarter
              FROM swift_codes WHERE swift_code = $1`
    err := r.db.QueryRow(query, swiftCode).Scan(&sc.SwiftCode, &sc.BankName, &sc.Address, &sc.CountryISO2, &sc.CountryName, &sc.IsHeadquarter)
    if err != nil {
        return SwiftCode{}, nil, err
    }

    var branches []SwiftCode
    if sc.IsHeadquarter {
        branchQuery := `SELECT swift_code, bank_name, address, country_iso2, country_name, is_headquarter
                       FROM swift_codes WHERE LEFT(swift_code, 8) = LEFT($1, 8) AND swift_code != $1`
        rows, err := r.db.Query(branchQuery, swiftCode)
        if err != nil {
            return sc, nil, err
        }
        defer rows.Close()
        for rows.Next() {
            var branch SwiftCode
            if err := rows.Scan(&branch.SwiftCode, &branch.BankName, &branch.Address, &branch.CountryISO2, &branch.CountryName, &branch.IsHeadquarter); err != nil {
                return sc, nil, err
            }
            branches = append(branches, branch)
        }
    }
    return sc, branches, nil
}

func (r *Repository) GetSwiftCodesByCountry(countryISO2 string) ([]SwiftCode, error) {
    query := `SELECT swift_code, bank_name, address, country_iso2, country_name, is_headquarter
              FROM swift_codes WHERE country_iso2 = $1`
    rows, err := r.db.Query(query, strings.ToUpper(countryISO2))
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var swiftCodes []SwiftCode
    for rows.Next() {
        var sc SwiftCode
        if err := rows.Scan(&sc.SwiftCode, &sc.BankName, &sc.Address, &sc.CountryISO2, &sc.CountryName, &sc.IsHeadquarter); err != nil {
            return nil, err
        }
        swiftCodes = append(swiftCodes, sc)
    }
    return swiftCodes, nil
}

func (r *Repository) DeleteSwiftCode(swiftCode string) error {
    query := DELETE FROM swift_codes WHERE swift_code = $1
    result, err := r.db.Exec(query, swiftCode)
    if err != nil {
        return err
    }
    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return sql.ErrNoRows
    }
    return nil
}
File: internal/api/handlers.go
go
package api

import (
    "net/http"
    "strings"
    "github.com/YourUsername/swift-code-api/internal/db"
    "github.com/gin-gonic/gin"
)

type Handler struct {
    repo *db.Repository
}

func NewHandler(repo *db.Repository) *Handler {
    return &Handler{repo: repo}
}

func (h *Handler) GetSwiftCode(c *gin.Context) {
    swiftCode := c.Param("swiftCode")
    sc, branches, err := h.repo.GetSwiftCode(swiftCode)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "SWIFT code not found"})
        return
    }

    response := gin.H{
        "swiftCode":    sc.SwiftCode,
        "bankName":     sc.BankName,
        "address":      sc.Address,
        "countryISO2":  sc.CountryISO2,
        "countryName":  sc.CountryName,
        "isHeadquarter": sc.IsHeadquarter,
    }
    if sc.IsHeadquarter {
        response["branches"] = branches
    }
    c.JSON(http.StatusOK, response)
}

func (h *Handler) GetSwiftCodesByCountry(c *gin.Context) {
    countryISO2 := strings.ToUpper(c.Param("countryISO2"))
    swiftCodes, err := h.repo.GetSwiftCodesByCountry(countryISO2)
    if err != nil || len(swiftCodes) == 0 {
        c.JSON(http.StatusNotFound, gin.H{"error": "No SWIFT codes found for country"})
        return
    }
    c.JSON(http.StatusOK, gin.H{
        "countryISO2":  countryISO2,
        "countryName":  swiftCodes[0].CountryName,
        "swiftCodes":   swiftCodes,
    })
}

func (h *Handler) AddSwiftCode(c *gin.Context) {
    var sc db.SwiftCode
    if err := c.ShouldBindJSON(&sc); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid request body"})
        return
    }
    sc.CountryISO2 = strings.ToUpper(sc.CountryISO2)
    sc.CountryName = strings.ToUpper(sc.CountryName)
    if err := h.repo.SaveSwiftCode(sc); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to save SWIFT code"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "SWIFT code added successfully"})
}

func (h *Handler) DeleteSwiftCode(c *gin.Context) {
    swiftCode := c.Param("swiftCode")
    if err := h.repo.DeleteSwiftCode(swiftCode); err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "SWIFT code not found"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"message": "SWIFT code deleted successfully"})
}
Replace YourUsername with your GitHub username.
File: internal/api/routes.go
go
package api

import "github.com/gin-gonic/gin"

func SetupRoutes(r *gin.Engine, h *Handler) {
    v1 := r.Group("/v1/swift-codes")
    {
        v1.GET("/:swiftCode", h.GetSwiftCode)
        v1.GET("/country/:countryISO2", h.GetSwiftCodesByCountry)
        v1.POST("/", h.AddSwiftCode)
        v1.DELETE("/:swiftCode", h.DeleteSwiftCode)
    }
}
Create a folder called tests in the swift-code-api folder.
Create a test file:
File: tests/handlers_test.go
go
package tests

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "github.com/YourUsername/swift-code-api/internal/api"
    "github.com/YourUsername/swift-code-api/internal/db"
    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
)

func TestAddSwiftCode(t *testing.T) {
    repo, _ := db.NewRepository("postgres://user:password@localhost:5432/swift_test?sslmode=disable")
    handler := api.NewHandler(repo)
    router := gin.Default()
    api.SetupRoutes(router, handler)

    sc := db.SwiftCode{
        SwiftCode:    "TESTUS33XXX",
        BankName:     "Test Bank",
        Address:      "123 Test St",
        CountryISO2:  "US",
        CountryName:  "UNITED STATES",
        IsHeadquarter: true,
    }
    body, _ := json.Marshal(sc)
    req, _ := http.NewRequest("POST", "/v1/swift-codes", bytes.NewBuffer(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    assert.Equal(t, http.StatusOK, w.Code)
    var response map[string]string
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "SWIFT code added successfully", response["message"])
}
Replace YourUsername with your GitHub username.
Create a file called Dockerfile in the swift-code-api folder:
dockerfile
FROM golang:1.20-alpine
WORKDIR /app
COPY . .
RUN go mod download
RUN go build -o main ./cmd/api
EXPOSE 8080
CMD ["./main"]
Create a file called docker-compose.yml in the swift-code-api folder:
yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      - DB_CONN=postgres://user:password@db:5432/swift?sslmode=disable
  db:
    image: postgres:13
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=swift
    volumes:
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
volumes:
  db-data:
Create a folder called sql in the swift-code-api folder.
Create a file called init.sql in the sql folder:
sql
CREATE TABLE swift_codes (
    swift_code VARCHAR(11) PRIMARY KEY,
    bank_name VARCHAR(255) NOT NULL,
    address VARCHAR(255) NOT NULL,
    country_iso2 CHAR(2) NOT NULL,
    country_name VARCHAR(100) NOT NULL,
    is_headquarter BOOLEAN NOT NULL
);

CREATE INDEX idx_country_iso2 ON swift_codes (country_iso2);
Update docker-compose.yml to include the SQL file:
yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      - DB_CONN=postgres://user:password@db:5432/swift?sslmode=disable
  db:
    image: postgres:13
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=swift
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
