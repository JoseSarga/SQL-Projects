# Music Database Normalization and Optimization Project

## Project Overview

This project focuses on designing, normalizing, and optimizing a relational database for managing musical artists, albums, tracks, and genres. The main goals include:

- Normalizing the database structure to eliminate redundancy and improve efficiency.
- Cleaning and validating data to ensure data integrity.
- Implementing indexes and constraints to enhance performance and maintain consistency.
- Creating triggers for error handling and logging database operations.

## Technologies Used

- SQLite
- SQL for database design, normalization, indexing, and data validation

## Database Schema Design

### Initial Tables

Before normalization, the database contained redundant and inconsistently structured data:

#### Artists Table (Before Normalization)

```sql
CREATE TABLE Artists(
    ArtistID INTEGER PRIMARY KEY,
    Name TEXT NOT NULL,
    BirthDate DATE,
    Genre TEXT  -- Redundant data
);
```

#### Albums Table (Before Normalization)

```sql
CREATE TABLE Albums(
    AlbumID INTEGER PRIMARY KEY,
    Title TEXT NOT NULL,
    ReleaseDate DATE,
    ArtistID INTEGER,
    Genre TEXT, -- Redundant data
    FOREIGN KEY(ArtistID) REFERENCES Artists(ArtistID)
);
```

#### Tracks Table (Before Normalization)

```sql
CREATE TABLE Tracks(
    TrackID INTEGER PRIMARY KEY,
    Title TEXT NOT NULL,
    Duration INTEGER,
    AlbumID INTEGER,
    ArtistGenre TEXT, -- Transitive dependency
    FOREIGN KEY(AlbumID) REFERENCES Albums(AlbumID)
);
```

### Normalization Process

To eliminate redundancy and transitive dependencies, we introduced a `Genres` table and restructured existing tables:

#### Genres Table

```sql
CREATE TABLE Genres(
    GenreID INTEGER PRIMARY KEY,
    Name TEXT NOT NULL
);
```

#### Updated Artists Table

```sql
CREATE TABLE Artists(
    ArtistID INTEGER PRIMARY KEY,
    Name TEXT NOT NULL,
    BirthDate DATE NOT NULL,
    GenreID INTEGER,
    FOREIGN KEY(GenreID) REFERENCES Genres(GenreID),
    CHECK (BirthDate <= DATE('now','-18 years'))
);
```

#### Updated Albums Table

```sql
CREATE TABLE Albums(
    AlbumID INTEGER PRIMARY KEY,
    Title TEXT NOT NULL,
    ReleaseDate DATE,
    ArtistID INTEGER,
    FOREIGN KEY(ArtistID) REFERENCES Artists(ArtistID),
    CHECK (Title IS NOT NULL)
);
```

#### Updated Tracks Table

```sql
CREATE TABLE Tracks(
    TrackID INTEGER PRIMARY KEY,
    Title TEXT NOT NULL,
    Duration INTEGER CHECK(Duration>0),
    AlbumID INTEGER,
    FOREIGN KEY(AlbumID) REFERENCES Albums(AlbumID),
    CHECK (Title IS NOT NULL)
);
```

### Indexing for Performance Optimization

```sql
CREATE INDEX idx_artist_name ON Artists(Name);
CREATE INDEX idx_album_title ON Albums(Title);
CREATE INDEX idx_track_title ON Tracks(Title);
CREATE INDEX idx_genre_name ON Genres(Name);
```

## Data Cleaning and Transformation

To ensure data integrity, we performed the following validations and transformations:

1. **Removing Invalid Foreign Key References**

```sql
DELETE FROM Artists_Temp WHERE GenreID NOT IN (SELECT GenreID FROM Genres_Temp);
DELETE FROM Albums_Temp WHERE ArtistID NOT IN (SELECT ArtistID FROM Artists_Temp);
DELETE FROM Tracks_Temp WHERE AlbumID NOT IN (SELECT AlbumID FROM Albums_Temp);
```

2. **Cleaning Artists Data**

```sql
CREATE TABLE Artists_Cleaned AS SELECT * FROM Artists_Temp;
DELETE FROM Artists_Cleaned WHERE Name IS NULL OR Name = '';
DELETE FROM Artists_Cleaned WHERE ROWID NOT IN (SELECT MIN(ROWID) FROM Artists_Cleaned GROUP BY ArtistID);
UPDATE Artists_Cleaned SET Name = TRIM(Name);
```

3. **Cleaning and Formatting Dates**

```sql
ALTER TABLE Albums ADD COLUMN ReleaseDateFormatted DATE;
UPDATE Albums
SET ReleaseDateFormatted =
    CASE
        WHEN LENGTH(SUBSTR(ReleaseDate, -2)) = 2 THEN
            CASE
                WHEN CAST(SUBSTR(ReleaseDate, -2) AS INTEGER) < 24 THEN
                    '20' || SUBSTR(ReleaseDate, -2) || '-' ||
                    SUBSTR('0' || REPLACE(SUBSTR(ReleaseDate, 1, INSTR(ReleaseDate, '/') - 1), '/', ''), -2) || '-' ||
                    SUBSTR('0' || REPLACE(SUBSTR(ReleaseDate, INSTR(ReleaseDate, '/') + 1, INSTR(SUBSTR(ReleaseDate, INSTR(ReleaseDate, '/') + 1), '/') - 1), '/', ''), -2)
                ELSE
                    '19' || SUBSTR(ReleaseDate, -2) || '-' ||
                    SUBSTR('0' || REPLACE(SUBSTR(ReleaseDate, 1, INSTR(ReleaseDate, '/') - 1), '/', ''), -2) || '-' ||
                    SUBSTR('0' || REPLACE(SUBSTR(ReleaseDate, INSTR(ReleaseDate, '/') + 1, INSTR(SUBSTR(ReleaseDate, INSTR(ReleaseDate, '/') + 1), '/') - 1), '/', ''), -2)
            END
        ELSE
            NULL
    END
WHERE ReleaseDate IS NOT NULL;
```

## Error Handling and Logging

### Error Handling via Triggers

```sql
CREATE TRIGGER error_handling_insert
BEFORE INSERT ON Albums
FOR EACH ROW
WHEN NOT EXISTS (SELECT 1 FROM Artists WHERE ArtistID = NEW.ArtistID)
BEGIN
    SELECT RAISE(ABORT, 'Invalid ArtistID');
END;
```

### Operation Logging

```sql
CREATE TABLE OperationLog (
    LogID INTEGER PRIMARY KEY,
    OperationType TEXT,
    TableName TEXT,
    RecordID INTEGER,
    Timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TRIGGER log_insert
AFTER INSERT ON Artists
FOR EACH ROW
BEGIN
    INSERT INTO OperationLog (OperationType, TableName, RecordID)
    VALUES ('INSERT', 'Artists', NEW.ArtistID);
END;
```

## Conclusion

This project demonstrates how to effectively design, normalize, and optimize a relational database while implementing data validation, cleaning, indexing, and error handling mechanisms. The final structure ensures minimal redundancy, improved query performance, and enhanced data integrity.

