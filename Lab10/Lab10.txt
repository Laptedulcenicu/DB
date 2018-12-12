--ex1. Sa se modifice declansatorul inregistrare_noua, in asa fel, incat in cazul actualizarii auditoriului sa apara mesajul de 
--informare, care, in afara de disciplina si ora, va afisa codul grupei afectate, ziua, blocul, auditoriul vechi si auditoriul nou.

USE universitatea;
GO
DROP TRIGGER IF EXISTS inregistrar_noua 
GO
CREATE TRIGGER inregistrar_noua ON plan_studii.orarul
AFTER UPDATE
AS SET NOCOUNT ON
IF UPDATE(Auditoriu)
SELECT 'Lectia la disciplina ' + UPPER(plan_studii.discipline.Disciplina)+ ', a grupei ' + grupe.Cod_Grupa +
		', in ziua de ' + CAST(inserted.Zi as VARCHAR(5)) + ', care era preconizata la ora ' + CAST(inserted.Ora as VARCHAR(5))
		+ ', a fost transferata in aula ' + CAST(inserted.Auditoriu as VARCHAR(5)) + ', Blocul '+
		CAST(inserted.Bloc as VARCHAR(5)) + '. Auditoriul vechi: ' + CAST(deleted.Auditoriu as VARCHAR(5))+
		', Auditoriul nou: ' + CAST(inserted.Auditoriu as VARCHAR(5))
FROM inserted,deleted, plan_studii.discipline, grupe
WHERE deleted.Id_Disciplina = plan_studii.discipline.Id_Disciplina
AND inserted.Id_Grupa = grupe.Id_Grupa
GO
UPDATE plan_studii.orarul set Auditoriu=510 where Id_Profesor=108

--2. Sa se creeze declansatorul, care ar asigura popularea corecta (consecutiva) a tabelelor studenti si studenti_reusita, si ar permite evitarea erorilor la nivelul cheilor exteme.*/
USE universitatea;
GO
IF OBJECT_ID('Inregistrare2', 'TR') IS NOT NULL
DROP TRIGGER Inregistrare2;
GO
CREATE TRIGGER Inregistrare2 ON studenti
INSTEAD OF INSERT
AS
SET NOCOUNT ON
DECLARE @Id_Student int
SELECT @Id_Student = Id_Student FROM inserted
INSERT INTO studenti.studenti_reusita(Id_Disciplina, Id_Grupa,Id_Student,Id_Profesor,Data_Evaluare,Tip_Evaluare,Nota) 
	VALUES (101,2,@Id_Student,105,'2018-12-04','Testul 1',8.00);
INSERT INTO studenti.studenti
SELECT * FROM inserted
GO
select * from studenti.studenti
INSERT INTO studenti.studenti VALUES(1,'Lastname','Firstname','1998-01-01','mun. Chisinau')


/*ex3. Sa se creeze un declansator, care ar interzice micsorarea notelor in tabelul studenti_reusita si
modificarea valorilor campului Data_Evaluare, unde valorile acestui camp sunt nenule.
Declansatorul trebuie sa se lanseze, numai daca sunt afectate datele studentilor din grupa
"CIB171". Se va afisa un mesaj de avertizare in cazul tentativei de a incalca constrangerea.*/
use universitatea
go
if OBJECT_ID('inregistrare3','TR') is not null 
	drop trigger inregistrare3
	go
CREATE TRIGGER inregistrare3 ON studenti_reusita
AFTER UPDATE
AS
SET NOCOUNT ON
 IF UPDATE(NOTA)
DECLARE @ID_GRUPA INT = (select Id_Grupa from grupe where Cod_Grupa='CIB171')
IF (SELECT AVG(NOTA) FROM deleted WHERE Id_Grupa=@ID_GRUPA AND NOTA IS NOT NULL)>(SELECT AVG(NOTA) FROM inserted WHERE Id_Grupa=@ID_GRUPA AND NOTA IS NOT NULL)
BEGIN
PRINT('Nu se permite miscrorarea notelor pentru grupa CIB171')
ROLLBACK TRANSACTION
END
if UPDATE(Data_Evaluare)
		begin 
		PRINT 'Tentativa de modificare a campului Data_Evaluare a esuat'
		ROLLBACK;
	end
	go
UPDATE studenti_reusita SET Nota=nota-1 WHERE Id_Grupa= (select Id_Grupa from grupe where Cod_Grupa='CIB171')
UPDATE studenti_reusita SET Data_Evaluare='2018-01-25' WHERE Id_Grupa= (select Id_Grupa from grupe where Cod_Grupa='CIB171')

--ex4. Sa se creeze un declansator DDL care ar interzice modificarea coloanei Id_Disciplina in tabelele bazei de date universitatea cu afisarea mesajului respectiv.
USE universitatea
GO
IF EXISTS (SELECT * FROM sys.triggers WHERE parent_class=0 AND name='inregistrare4')
DROP TRIGGER inregistrare4 ON DATABASE;
GO
CREATE TRIGGER inregistrare4
ON DATABASE
FOR ALTER_TABLE
AS
SET NOCOUNT ON
DECLARE @Id_Disciplina int
SELECT @Id_Disciplina=EVENTDATA().value('(/EVENT_INSTANCE/AlterTableActionList/*/Columns/Name)[1]', 'nvarchar(max)')
IF @Id_Disciplina='Id_Disciplina'
BEGIN
PRINT('Este interzisa modificarea coloanei Id_Disciplina');
ROLLBACK;
END
go
use universitatea
go 
alter table discipline alter column Id_Disciplina smallint

--ex5. Sa se creeze un declansator DDL care ar interzice modificarea schemei bazei de date in afara orelor de lucru.
use universitatea;
GO
DROP TRIGGER if exists inregistrar5 ON DATABASE
GO
CREATE TRIGGER inregistrar5 ON DATABASE 
FOR ALTER_TABLE
AS
SET NOCOUNT ON
DECLARE @TimpActual DATETIME
DECLARE @Inceput DATETIME
DECLARE @Sfarsit DATETIME
DECLARE @A FLOAT
DECLARE @B FLOAT
SELECT @TimpActual=GETDATE()
SELECT @Inceput ='2018-12-04 8:00'
SELECT @Sfarsit = '2018-12-04 15:00'
select @A=(cast (@TimpActual as float)-floor(cast(@TimpActual as float)))-
          (cast(@Inceput as float)-floor(cast(@Inceput as float))),
       @B=(cast(@TimpActual as float)-floor(cast(@TimpActual as float)))-
	      (cast(@Sfarsit as float)-floor(cast(@Sfarsit as float)))
IF @A<0 or @B>0
BEGIN
Print('Este interzisa modificarea schemei bazei de date in afara orelor de lucru')
ROLLBACK;
END
go

--ex6. Sa se creeze un declansator DDL care, la modificarea proprietatilor coloanei Id_Profesor dintr-un tabel, ar face schimbari asemanatoare in mod automat in restul tabelelor.
USE universitatea
GO
IF EXISTS (SELECT * FROM sys.triggers WHERE parent_class=0 AND name='inregistrare6')
DROP TRIGGER inregistrare6 ON DATABASE;
GO
CREATE TRIGGER inregistrare6 ON DATABASE
FOR ALTER_TABLE
AS
SET NOCOUNT ON
DECLARE @id int
DECLARE @int_I varchar(500)
DECLARE @int_M varchar(500)
DECLARE @den_T varchar(50)
SELECT @id=EVENTDATA().
value('(/EVENT_INSTANCE/AlterTableActionList/*/Columns/Name)[1]','nvarchar(max)')
IF @id = 'Id_Profesor'
BEGIN
SELECT @int_I = EVENTDATA().value('(/EVENT_INSTANCE/TSQLCommand/CommandText)[1]','nvarchar(max)')
SELECT @den_T = EVENTDATA().value('(/EVENT_INSTANCE/ObjectName)[1]','nvarchar(max)')
SELECT @int_M = REPLACE(@int_I, @den_T, 'studenti_reusita');EXECUTE (@int_M)
SELECT @int_M = REPLACE(@int_I, @den_T, 'grupe');EXECUTE (@int_M)
SELECT @int_M = REPLACE(@int_I, @den_T, 'profesori');EXECUTE (@int_M)
PRINT 'Datele au fost modificate automat'
END
go
alter table profesori alter column Id_Profesor smallint