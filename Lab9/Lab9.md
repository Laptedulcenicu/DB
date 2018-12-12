Sarcini practice :
Ex.1
Sa se creeze proceduri stocate in baza exercitiilor (2 exercitii) din capitolul 4. Parametrii de intrare trebuie sa corespunda criteriilor din clauzele WHERE ale exercitiilor respective.

--Problema 7. In ce grupa(Cod_grupa invata studentii care locuiesc pe strada 31 August?
use universitatea
go
drop procedure if exists A2
go
Create procedure A2
@Adresa varchar(50) ='%31 August%'

as
begin
select DISTINCT Cod_Grupa 
from grupe g
inner join reusita r
on g.Id_Grupa=r.Id_Grupa
inner join student s
on r.ID_Student= s.ID_Student
where Adresa_Postala_Student Like @Adresa
end
exec A2;

Ex.2
Sa se creeze o procedura stocata, care nu are niciun parametru de intrare si poseda un parametru de iesire.
 Parametrul de iesire trebuie sa returneze numarul de studenti, care nu au sustinut eel putin o forma de evaluare (nota mai mica de 5 sau valoare NULL).

DROP PROCEDURE IF EXISTS A2;
GO

CREATE PROCEDURE A2
   @nr_de_studenti SMALLINT = NULL OUTPUT
AS
  
SELECT @nr_de_studenti =  COUNT(DISTINCT Id_student) 
FROM reusita
WHERE Nota < 5 or Nota = NULL

DECLARE @nr_de_studenti SMALLINT
EXEC Lab9_ex2 @nr_de_studenti OUTPUT
PRINT 'Nr de studenti ce nu au sustinut cel putin o forma de evaluare = ' + cast(@nr_de_studenti as VARCHAR(3))


Ex.3
Sa se creeze o procedura stocata, care ar insera in baza de date informatii despre un student nou. 
In calitate de parametri de intrare sa serveasca datele personale ale studentului nou si Cod_Grupa.
 Sa se genereze toate intrarile-cheie necesare in tabelul studenti_reusita. 
Notele de evaluare sa fie inserate ca NULL.

DROP PROCEDURE IF EXISTS A3
GO

CREATE PROCEDURE A3 
@numeStudent VARCHAR(50),
@prenumeStudent VARCHAR(50),
@data DATE,
@adresa VARCHAR(500),
@cod_grupa CHAR(6)

AS
INSERT INTO student 
VALUES (07, @numeStudent, @prenumeStudent, @data, @adresa)
INSERT INTO reusita
VALUES (07, 100, 100 , 
         (SELECT Id_Grupa FROM grupe WHERE Cod_Grupa = @cod_grupa), 'examen', NULL, '2018-11-25')

		 exec A3  'Laptedulce', 'Nicusor', '1997-08-10',' mun.Chisinau', 'FAF172'

select * from student

Ex.4
Fie ca un profesor se elibereaza din functie la mijlocul semestrului.
 Sa se creeze o procedura stocata care ar reatribui inregistrarile din tabelul studenti_reusita unui alt profesor.
 Parametri de intrare: numele si prenumele profesorului vechi, numele si prenumele profesorului nou, disciplina.
 in cazul in care datele inserate sunt incorecte sau incomplete, sa se afi~eze un mesaj de avertizare.



 DROP PROCEDURE IF EXISTS ChimbProfesori
GO
CREATE PROCEDURE SchimbProfesori
@Nume_prof VARCHAR(60),
@Pren_prof VARCHAR(60),
@NumeP_nou VARCHAR(60),
@PrenP_nou VARCHAR(60),
@Disciplina VARCHAR(20),
@Error INT = NULL
AS
IF(( Select plan_studii.discipline.Id_Disciplina 
     From plan_studii.discipline 
	 Where Disciplina = @Disciplina) IN (Select distinct studenti.studenti_reusita.Id_Disciplina 
										 From studenti.studenti_reusita 
										 Where Id_Profesor = (Select cadre_didactice.profesori.Id_Profesor 
										                      From cadre_didactice.profesori 
															  Where Nume_Profesor = @Nume_prof AND Prenume_Profesor = @Pren_prof)))
UPDATE studenti.studenti_reusita
SET Id_Profesor =  (Select Id_Profesor
					From cadre_didactice.profesori
					Where Nume_Profesor = @NumeP_nou AND   Prenume_Profesor = @PrenP_nou)

WHERE Id_Profesor = (Select Id_profesor
					 FROM cadre_didactice.profesori
     			     WHERE Nume_Profesor = @Nume_prof AND Prenume_Profesor = @Pren_prof)
					 
 SET @Error = @@ERROR
 IF @Error <>0
 BEGIN
 RAISERROR ('ERROR : The  inserted   data  is  either incorrect  or  incomplete ' ,10 ,1)
 END



Ex.5
Sa se creeze o procedura stocata care ar forma o lista cu primii 3 cei mai buni studenti la o disciplina, si acestor studenti sa le fie marita nota la examenul final cu un punct (nota maximala posibila este 10).
 In calitate de parametru de intrare, va servi denumirea disciplinei.
 Procedura sa returneze urmatoarele campuri: Cod_Grupa, Nume_Prenume_Student, Disciplina, Nota_ Veche, Nota_Noua.


DROP PROCEDURE IF EXISTS ex5
GO
CREATE PROCEDURE ex5 
@disciplina VARCHAR(50)
AS
DECLARE @stud_list TABLE (Id_Student int, Media float)
INSERT INTO @stud_list
	SELECT TOP (3) Id_Student, AVG(cast (Nota as float)) as Media
	FROM reusita, disciplina
	WHERE disciplina.Id_Disciplina = reusita.Id_Disciplina
	AND Disciplina = @disciplina
	GROUP BY reusita.Id_Student
	ORDER BY Media desc		

SELECT cod_grupa, student.Id_Student, CONCAT(nume_student, ' ', Prenume_Student) as Nume, Disciplina, nota AS Nota_Veche, iif(nota > 9, 10, nota + 1) AS Nota_Noua 
	FROM reusita, disciplina, grupe, student WHERE disciplina.id_disciplina = reusita.id_disciplina
	AND grupe.Id_Grupa = reusita.Id_Grupa
	AND  student.Id_Student = reusita.Id_Student
	AND student.Id_Student in (select Id_Student from @stud_list)
	AND Disciplina = @disciplina
	AND Tip_Evaluare = 'Examen'

DECLARE @id_dis SMALLINT = (SELECT  Id_Disciplina  FROM disciplina WHERE   Disciplina = @disciplina)

UPDATE reusita SET Nota = (CASE WHEN nota >= 9 THEN 10 ELSE nota + 1 END)
WHERE Tip_Evaluare = 'Examen' AND Id_Disciplina = @id_dis AND Id_Student in (select Id_Student from @stud_list)
go

execute ex5 @disciplina = 'Informatica aplicata'


Ex. 6
Sa se creeze functii definite de utilizator in baza exercitiilor (2 exercitii) din capitolul 4.
 Parametrii de intrare trebuie sa corespunda criteriilor din clauzele WHERE ale exercitiilor respective.

--Problema 7. In ce grupa(Cod_grupa invata studentii care locuiesc pe strada 31 August?

drop function if exists A2
go
Create function A2 ( @Adresa varchar(50))
returns table
as
RETURN

(select DISTINCT Cod_Grupa 
from grupe g
inner join reusita r
on g.Id_Grupa=r.Id_Grupa
inner join student s
on r.ID_Student= s.ID_Student
where Adresa_Postala_Student Like @Adresa)


select * from A2('%31 August%')


--Problema 15. Gasiti numele si prenumele studentilor, care au sustinut examen atit la profesorul Ion ,
--cit si la profesorul George in anul 2019(folositi pentru nume clauza Like)

use universitatea
go
drop function if exists A2
go
Create function A2(@NumeUnu varchar(10),@NumeDoi varchar(10),@Year date,@Year2 date,@evaluarea varchar(10))
returns table
as
return
(select  distinct Nume_Student, Prenume_Student 
from student s
inner join reusita r
on s.ID_Student= r.ID_Student
inner join cadre_didactice.profesori p
on r.Id_Profesor= p.Id_Profesor
where Prenume_Profesor  like @NumeUnu 
 and Tip_Evaluare = @evaluarea and Data_Evaluare between @Year and  @Year2
 intersect
 select  distinct Nume_Student, Prenume_Student 
from student s
inner join reusita r
on s.ID_Student= r.ID_Student
inner join cadre_didactice.profesori p
on r.Id_Profesor= p.Id_Profesor
where Prenume_Profesor  like @NumeDoi
 and Tip_Evaluare = @evaluarea and  Data_Evaluare between @Year and  @Year2)


select * from A2('Ion','George','2017-01-01','2017-12-31','Examen')


Ex.7
Sa se scrie functia care ar calcula varsta studentului. Sa se defineasca urmatorul format al functiei: <nume_functie>(<Data_Nastere_Student>).



DROP FUNCTION IF EXISTS Lab9_ex7
GO

CREATE FUNCTION Lab9_ex7 (@data_nasterii DATE )
RETURNS INT
 BEGIN
 DECLARE @varsta INT
 SELECT @varsta = (SELECT (YEAR(GETDATE()) - YEAR(@data_nasterii) - CASE 
 						WHEN (MONTH(@data_nasterii) > MONTH(GETDATE())) OR (MONTH(@data_nasterii) = MONTH(GETDATE()) AND  DAY(@data_nasterii)> DAY(GETDATE()))
						THEN  1
						ELSE  0
						END))
 RETURN @varsta
 END

 select dbo.Lab9_ex7 ('1997-10-08') as Virsta



Ex.8
Sa se creeze o functie definita de utilizator, care ar returna datele referitoare la reusita unui student.
 Se defineste urmatorul format al functiei : < nume_functie > (<Nume_Prenume_Student>). Sa fie afisat tabelul cu urmatoarele campuri: Nume_Prenume_Student, Disticplina, Nota, Data_Evaluare.

DROP FUNCTION IF EXISTS Lab9_ex8
GO

CREATE FUNCTION Lab9_ex8 (@nume_prenume_s VARCHAR(50))
RETURNS TABLE 
AS
RETURN
(SELECT Nume_Student + ' ' + Prenume_Student as Student, Disciplina, Nota, Data_Evaluare
 FROM studentiS, disciplineS, reusitaS
 WHERE studentiS.Id_Student = reusitaS.Id_Student
 AND disciplineS.Id_Disciplina = reusitaS.Id_Disciplina 
 AND Nume_Student + ' ' + Prenume_Student = @nume_prenume_s )


Ex.8
Sa se creeze o functie definita de utilizator, care ar returna datele referitoare la reusita unui student. Se defineste urmatorul format al functiei : < nume_functie > (<Nume_Prenume_Student>). Sa fie afisat tabelul cu urmatoarele campuri: Nume_Prenume_Student, Disticplina, Nota, Data_Evaluare.



DROP FUNCTION IF EXISTS Lab9_ex8
GO

CREATE FUNCTION Lab9_ex8 (@nume_prenume_s VARCHAR(50))
RETURNS TABLE 
AS
RETURN
(SELECT Nume_Student + ' ' + Prenume_Student as Student, Disciplina, Nota, Data_Evaluare
 FROM student, disciplina, reusita
 WHERE student.Id_Student = reusita.Id_Student
 AND disciplina.Id_Disciplina = reusita.Id_Disciplina 
 AND Nume_Student + ' ' + Prenume_Student = @nume_prenume_s )

 select * from dbo.Lab9_ex8 ('Dan David')

Ex.9
Se cere realizarea unei functii definite de utilizator, care ar gasi cel mai sarguincios sau cel mai slab student dintr-o grupa. Se defineste urmatorul format al functiei: <nume_functie> (<Cod_Grupa>, <is_good>). Parametrul <is_good> poate accepta valorile "sarguincios" sau "slab", respectiv. Functia sa returneze un tabel cu urmatoarele campuri Grupa, Nume_Prenume_Student, Nota Medie , is_good.
 Nota Medie sa fie cu precizie de 2 zecimale.


DROP FUNCTION IF EXISTS Lab9_ex9
GO

CREATE FUNCTION Lab9_ex9 (@cod_grupa VARCHAR(10), @is_good VARCHAR(20))
RETURNS @Test Table (Cod_Grupa varchar(10), Student varchar (100), Media decimal(4,2), Reusita varchar(20))
AS
begin

if @is_good = 'sarguincios'
begin
insert into @Test

SELECT top (1) Cod_Grupa, Nume_Student + ' ' + Prenume_Student as Student,
		 CAST(AVG( Nota * 1.0) as decimal (4,2)) as Media, @is_good
 FROM grupe,student, reusita
 WHERE grupe.Id_Grupa = reusita.Id_Grupa
 AND student.Id_Student = reusita.Id_Student
 AND Cod_Grupa = @cod_grupa
 GROUP BY Cod_Grupa, Nume_Student, Prenume_Student
 Order by Media desc
 end
 else

 begin 
 insert into @Test
SELECT top (1) Cod_Grupa, Nume_Student + ' ' + Prenume_Student as Student,
		 CAST(AVG( Nota * 1.0) as decimal (4,2)) as Media, @is_good
 FROM grupe,student, reusita
 WHERE grupe.Id_Grupa = reusita.Id_Grupa
 AND student.Id_Student = reusita.Id_Student
 AND Cod_Grupa = @cod_grupa
 GROUP BY Cod_Grupa, Nume_Student, Prenume_Student
 Order by Media 
 
end


 RETURN 
 end




select * from dbo.Lab9_ex9 ('TI171','sarguincios')
 select * from dbo.Lab9_ex9 ('CIB171','sarguincios')
 select * from dbo.Lab9_ex9 ('INF171','sarguincios')








