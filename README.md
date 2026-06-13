<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>АФО</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
body { background:#000; color:#f00; font-family:Arial; text-align:center; margin:0; padding:20px; }
button { background:#800000; color:#fff; border:none; padding:12px; cursor:pointer; margin:8px 0; width:100%; font-size:16px; border-radius: 5px; }
button:hover { background: #a00; }
textarea, input { width:100%; background:#222; color:#fff; border:1px solid #444; padding:10px; margin:5px 0; font-size:16px; box-sizing: border-box; }
.post { border-left:4px solid #f00; padding:15px; margin:20px 0; background:#151515; text-align:left; }
.comment { background:#222; padding:8px; margin:8px 0; border-bottom: 1px solid #333; font-size: 14px; color: #ccc; }
.delete-btn { background:#f00; color:#fff; border:none; padding:2px 8px; cursor:pointer; float: right; font-size: 12px; }
</style>

<!-- Firebase -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-firestore-compat.js"></script>

</head>
<body>

<h1 id="title">Добро пожаловать в АФО</h1>

<div id="login">
  <button onclick="loginAuthor()">Войти как автор</button>
  <button onclick="loginUser()">Войти как пользователь</button>
</div>

<div id="authorPanel" style="display:none">
  <h2>Новая статья</h2>
  <input id="postTitle" placeholder="Заголовок">
  <textarea id="postText" placeholder="Текст статьи"></textarea>
  <button onclick="publish()">Опубликовать</button>
</div>

<h2 id="postsHeader" style="display:none">Лента новостей</h2>
<div id="posts" style="display:none"></div>

<script>
  // === Firebase Config ===
  const firebaseConfig = {
    apiKey: "AIzaSyB3Ihim9lcyL5AKfXi055uMw_1--u7vYMQ",
    authDomain: "liou-fea92.firebaseapp.com",
    projectId: "liou-fea92",
    storageBucket: "liou-fea92.appspot.com",
    messagingSenderId: "1009858434231",
    appId: "1:1009858434231:web:ddb05c5602188bf0012932",
    measurementId: "G-5G05YL7SVG"
  };

  firebase.initializeApp(firebaseConfig);
  const db = firebase.firestore();

  let isAuthor = false;

  // === Авторизация ===
  window.loginAuthor = () => {
    const code = prompt("Введите код автора:");
    if (code === "120812") {
      isAuthor = true;
      showApp("Режим автора");
      document.getElementById("authorPanel").style.display = "block";
    } else {
      alert("Неверный код");
    }
  };

  window.loginUser = () => {
    isAuthor = false;
    showApp("Режим пользователя");
  };

  function showApp(titleText) {
    document.getElementById("login").style.display = "none";
    document.getElementById("title").innerText = titleText;
    document.getElementById("postsHeader").style.display = "block";
    document.getElementById("posts").style.display = "block";
    loadPosts();
  }

  // === Публикация статьи ===
  window.publish = async () => {
    const title = document.getElementById("postTitle").value.trim();
    const text = document.getElementById("postText").value.trim();
    if (!title || !text) return alert("Заполни всё");
    try {
      await db.collection("posts").add({
        title,
        text,
        comments: [],
        likes: 0,
        author: "admin", // автор
        time: Date.now()
      });
      document.getElementById("postTitle").value = "";
      document.getElementById("postText").value = "";
    } catch(e) {
      alert("Ошибка: " + e.message);
    }
  };

  // === Загрузка постов ===
  function loadPosts() {
    db.collection("posts").orderBy("time", "desc").onSnapshot(snap => {
      const postsDiv = document.getElementById("posts");
      postsDiv.innerHTML = "";

      snap.forEach(docSnap => {
        const p = docSnap.data();
        const id = docSnap.id;

        const div = document.createElement("div");
        div.className = "post";

        // Комментарии: автор может удалять чужие
        let commentsHTML = (p.comments || []).map(c => `
          <div class="comment">
            ${c.text} 
            ${isAuthor && c.user !== "admin" ? `<button class="delete-btn" onclick="deleteComment('${id}','${c.text}')">✖</button>` : ''}
          </div>
        `).join("");

        div.innerHTML = `
          <h3>${p.title}</h3>
          <p>${p.text}</p>
          <button onclick="like('${id}', ${p.likes})">❤️ ${p.likes}</button>
          <div class="comments-section">${commentsHTML}</div>
          <textarea id="c${id}" placeholder="Ваш комментарий..."></textarea>
          <button onclick="comment('${id}')">Отправить</button>
          ${isAuthor && p.author === "admin" ? `<button style="background:#444; margin-top:10px;" onclick="deletePost('${id}')">Удалить статью</button>` : ''}
        `;
        postsDiv.appendChild(div);
      });
    });
  }

  // === Лайки ===
  window.like = async (id, currentLikes) => {
    await db.collection("posts").doc(id).update({ likes: (currentLikes || 0) + 1 });
  };

  // === Комментарии ===
  window.comment = async (id) => {
    const input = document.getElementById("c" + id);
    if (!input.value.trim()) return;
    const comment = { text: input.value.trim(), user: isAuthor ? "admin" : "user" };
    await db.collection("posts").doc(id).update({
      comments: firebase.firestore.FieldValue.arrayUnion(comment)
    });
    input.value = "";
  };

  // === Удаление комментария ===
  window.deleteComment = async (postId, commentText) => {
    const postRef = db.collection("posts").doc(postId);
    const postDoc = await postRef.get();
    if (!postDoc.exists) return;
    const comments = postDoc.data().comments || [];
    const commentToDelete = comments.find(c => c.text === commentText);
    if (commentToDelete) {
      await postRef.update({
        comments: firebase.firestore.FieldValue.arrayRemove(commentToDelete)
      });
    }
  };

  // === Удаление статьи (только свои) ===
  window.deletePost = async (postId) => {
    if (!isAuthor) return;
    if (confirm("Удалить статью?")) {
      await db.collection("posts").doc(postId).delete();
    }
  };
</script>

</body>
</html>