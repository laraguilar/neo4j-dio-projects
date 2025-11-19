# Atividade: Sistema de Recomendação de Música

## Descrição da Atividade
![WhatsApp Image 2025-11-18 at 21 24 21_e4d39c28](https://github.com/user-attachments/assets/55befcd7-1a8b-4c14-8cd0-9c7497f35292)


## Modelagem (arrows.app)
<img width="927" height="533" alt="Untitled graph (4)" src="https://github.com/user-attachments/assets/b5ec4390-1583-4265-9b26-fed00158e3a8" />

### Definindo constraints

- Id's únicos

      // Artista
        CREATE CONSTRAINT artist_id_unique IF NOT EXISTS
        FOR (a:Artist)
        REQUIRE a.id IS UNIQUE;
        
        // Usuário
        CREATE CONSTRAINT user_id_unique IF NOT EXISTS
        FOR (u:User)
        REQUIRE u.id IS UNIQUE;
        
        // Música
        CREATE CONSTRAINT music_id_unique IF NOT EXISTS
        FOR (m:Music)
        REQUIRE m.id IS UNIQUE;
        
        // Gênero
        CREATE CONSTRAINT genre_id_unique IF NOT EXISTS
        FOR (g:Genre)
        REQUIRE g.id IS UNIQUE;

- E-mail e CPF únicos (Usuário)

        CREATE CONSTRAINT user_email_unique IF NOT EXISTS
        FOR (u:User)
        REQUIRE u.email IS UNIQUE;
        
        CREATE CONSTRAINT user_cpf_unique IF NOT EXISTS
        FOR (u:User)
        REQUIRE u.cpf IS UNIQUE;


- Name único (Artista e Gênero)

        // Artista
        CREATE CONSTRAINT artist_name_unique IF NOT EXISTS
        FOR (a:Artist)
        REQUIRE a.name IS UNIQUE;
        
        // Gênero
        CREATE CONSTRAINT genre_name_unique IF NOT EXISTS
        FOR (g:Genre)
        REQUIRE g.name IS UNIQUE;

### Definindo as relações

Os id's devem ser substituidos pelo ID dos nós referentes.

- User - FOLLOW - Artist

        MATCH (u:User {id: $userId})
        MATCH (a:Artist {id: $artistId})
        MERGE (u)-[:FOLLOW]->(a);

- User - LISTEN - Music

        MATCH (u:User {id: $userId})
        MATCH (m:Music {id: $musicId})
        MERGE (u)-[:LISTEN {rating: $rating, nListen: $nListen}]->(m);

- Artist - RELEASE - Music

        MATCH (a:Artist {id: $artistId})
        MATCH (m:Music {id: $musicId})
        MERGE (a)-[:RELEASE {date: $date}]->(m);


- Music - HAS - Genre

        MATCH (m:Music {id: $musicId})
        MATCH (g:Genre {id: $genreId})
        MERGE (m)-[:HAS]->(g);

## Carregando dados

Os dados utilizados foi do [Spotify Global Music Dataset (2009–2025)](https://www.kaggle.com/datasets/wardabilal/spotify-global-music-dataset-20092025?resource=download).

Foi necessário aplicar um filtro na coluna de track_name e artist_genre para remover títulos com aspas duplas.


### Dados de Music, Artist e Genre

      LOAD CSV WITH HEADERS FROM 'file:///spotify_data_clean.csv' AS row
      CALL {
          WITH row
      
          // MUSIC
          MERGE (m:Music { id: row.track_id })
          SET m.title = row.track_name,
              m.time = row.track_duration_min,
              m.popularity = toInteger(row.track_popularity),
              m.explicit = (row.explicit = "true")
      
          // ARTIST
          MERGE (a:Artist { name: row.artist_name })
          SET a.popularity = toInteger(row.artist_popularity)
      
          // GENRE
          MERGE (g:Genre { name: row.artist_genres })
      
          // Artist - RELEASE - Music
          MERGE (a)-[:RELEASE]->(m)
      
          // Music - HAS - Genre
          MERGE (m)-[:HAS]->(g)
      
      }
      IN TRANSACTIONS OF 1000 ROWS;

### Dados de User
Através da IA, gerei uma planilha com dados de 10 usuários fictícios, relacionando-os à tabela de música.

      LOAD CSV WITH HEADERS FROM 'file:///user_track_listens.csv' AS row
      CALL {
          WITH row
      
          // MUSIC
          MERGE (m:Music { id: row.track_id })
      
          // USER
          MERGE (u:User { cpf: row.user_cpf, email: row.user_email })
          SET u.name = row.user_name,
              u.cpf = row.user_cpf,
              u.email = row.user_email
      
          // User - LISTEN - Music
          MERGE (u)-[r:LISTEN]->(m)
          SET r.rating = toInteger(row.rating_track),
              r.nListen = toInteger(row.n_listen)
      }
      IN TRANSACTIONS OF 1000 ROWS;

### Dados de User - FOLLOWS -> Artist

Através da IA, gerei uma nova planilha relacionando os usuários com os artistas que seguem

      LOAD CSV WITH HEADERS FROM 'file:///user_follows_artist.csv' AS row
      CALL {
          WITH row
      
          // USER
          MERGE (u:User { cpf: row.user_cpf, email: row.user_email })
      
          // ARTIST
          MERGE (a:Artist {id: row.artist_id})
      
          // User - FOLLOWS - Music
          MERGE (u)-[r:FOLLOWS]->(a)
      }
      IN TRANSACTIONS OF 1000 ROWS;


### Visualização do Schema 

<img width="712" height="601" alt="visualisation" src="https://github.com/user-attachments/assets/c98cefec-dd2a-4b7f-b692-ea7cf73fdfbb" />


## Resultados

### 1. Playlist Recomendada

A playlist recomendada busca pelas músicas que os "vizinhos" do usuário escutam. Isto é, baseado no que o usuário ouve, encontramos os usuários (vizinhos) que também ouvem aquelas músicas e a partir disso, encontramos o que os vizinhos escutam que o usuário alvo não escuta.

      
      WITH 'email_do_usuario@exemplo.com' AS targetEmail // define usuário alvo
      
      MATCH (me:User {email: targetEmail})-[:LISTEN]->(m:Music)<-[r1:LISTEN]-(other:User) // encontra usuário e seus vizinhos
      WHERE me <> other AND r1.rating >= 3 
      WITH me, other, count(m) AS overlap
      
      MATCH (other)-[r2:LISTEN]->(rec:Music) // encontra o que os vizinhos andaram ouvindo
      WHERE NOT (me)-[:LISTEN]->(rec) AND r2.rating >= 4 

      // retorna resultado
      RETURN rec.title AS Musica,
             rec.time AS Duracao,
             collect(DISTINCT other.name)[0..3] AS QuemTambemOuviu,
             count(other) AS Frequencia, 
             avg(r2.rating) AS NotaMedia
      ORDER BY Frequencia DESC, NotaMedia DESC
      LIMIT 10; 


<img width="1093" height="477" alt="image" src="https://github.com/user-attachments/assets/e5f719de-422b-40a9-8784-2ec0bd8632c1" />


### 2. Artistas Sugeridos

      WITH 'user1@example.com' AS targetEmail

      // 1. Descobre os gêneros favoritos (Top 3)
      MATCH (u:User {email: targetEmail})-[r:LISTEN]->(m:Music)-[:HAS]->(g:Genre)
      WITH u, g, count(r) AS genreScore
      ORDER BY genreScore DESC
      LIMIT 3
      
      // 2. Encontra artistas desses gêneros através das músicas
      MATCH (g)<-[:HAS]-(m2:Music)<-[:RELEASE]-(rec:Artist)
      WHERE NOT (u)-[:FOLLOWS]->(rec)
      
      // 3. O TRUQUE ESTÁ AQUI: Agregação
      // Ao usar 'collect', o Neo4j entende que deve agrupar as linhas pelo Artista
      RETURN rec.name AS ArtistaSugerido, 
             collect(DISTINCT g.name) AS GenerosEmComum, // Lista os gêneros sem repetir
             rec.popularity AS Popularidade,
             count(m2) AS FaixasNoGenero // Opcional: Mostra quantas músicas desse gênero o artista tem
      ORDER BY Popularidade DESC
      LIMIT 5;

<img width="1110" height="292" alt="image" src="https://github.com/user-attachments/assets/5646ea12-0ff8-4415-b6c5-5ada0f08e6c5" />

