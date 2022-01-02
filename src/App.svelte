<script>
  import { io } from "socket.io-client";

  const socket = io("https://e4e4-103-221-209-21.ngrok.io", {
    transports: ["websocket", "polling"],
  });

  let isHost = false,
    current = undefined,
    thisPeer = "",
    thisStream;

  let pc = {},
    candidates = [];

  // $: state = [...pc.map((e) => e.connectionState)];

  const createPeerConnection = (n) => {
    const rpc = new RTCPeerConnection({
      iceServers: [
        {
          urls: [
            "stun:stun1.l.google.com:19302",
            "stun:stun1.l.google.com:19302",
          ],
        },
      ],
      iceCandidatePoolSize: 10,
    });

    rpc.addStream(thisStream);

    rpc.onicecandidate = (e) => {
      if (e.candidate) {
        candidates.push(e.candidate);
      }
    };

    rpc.oniceconnectionstatechange = (e) =>
      (state = e.currentTarget.connectionState);

    rpc.ontrack = (e) => {
      console.log(`Track is here. :: ${e}`);
      const vid = document.createElement("video");
      vid.setAttribute("class", "video");
      vid.setAttribute("id", `remote-video-${n}`);
      vid.setAttribute("autoplay", "autoplay");
      vid.setAttribute("muted", "muted");
      vid.setAttribute("controls", "controls");
      console.log(vid);
      document.querySelector("main").append(vid);
    };

    return rpc;
  };

  const createOffer = (n) => {
    console.log(pc, n);
    pc[n]
      .createOffer({
        offerToReceiveAudio: 1,
        offerToReceiveVideo: 1,
      })
      .then((sdp) => {
        pc[n].setLocalDescription(sdp);
        send("SDP", sdp);
      })
      .catch(console.log);
  };

  const createAnswer = (n) => {
    pc[n]
      .createAnswer({
        offerToReceiveAudio: 1,
        offerToReceiveVideo: 1,
      })
      .then((sdp) => {
        pc[n].setLocalDescription(sdp);
        send("SDP", sdp);
      })
      .catch(console.log);
  };

  socket.on("connection::ok", (socket, host) => {
    console.log(`HOST :: ${host} :: ${socket}`);
    pc = { ...pc, [socket]: createPeerConnection(1) };
    thisPeer = socket;
    if (host) {
      isHost = host;
      setTimeout(() => createOffer(socket), 100);
    }
  });

  const send = (event, payload, to) => {
    // console.log({
    //   event,
    //    to: current,
    //    data: payload
    // });
    socket.emit(event, { to: current, data: payload });
  };

  socket.on("SDP", ({ from, data }) => {
    console.log(
      `SOCKET ON :: SDP ${isHost ? "PEER" : "REMOTE PEER"} ${
        data.type
      } :: ${from}`
    );
    current = from;

    if (isHost && data.type === "answer") {
      console.log(pc, thisPeer);
      pc[thisPeer].setRemoteDescription(new RTCSessionDescription(data));
      candidates.forEach((e) => send("CANDIDATE", e));
    } else if (!isHost && data.type === "offer") {
      console.log(pc, thisPeer);
      pc[thisPeer].setRemoteDescription(new RTCSessionDescription(data));
      createAnswer(thisPeer);
    }
  });

  socket.on("CANDIDATE", ({ from, data }) => {
    current = from;
    console.log("SOCKET ON :: CANDIDATE");
    data && pc[from].addIceCandidate(new RTCIceCandidate(data));
  });

  socket.on("is::new", (id, count) => {
    current = id;
    if (count > 2) {
      pc = { ...pc, [id]: createPeerConnection(Object.keys(pc).length) };
      createOffer(id);
    } else {
      createOffer(thisPeer);
    }
    console.log(pc);
  });

  socket.on("log", (data) => {
    console.log(`Connections :: ${data}`);
    // if (data < 3) {
    //   if (Object.values(pc)[0].connectionState === "new") {
    //     createOffer(Object.keys(pc)[0]);
    //   }
    // }
  });

  socket.on("error", (e) => console.info(e));

  let constraints = {
    audio: true,
    video: {
      width: { min: 1024, ideal: 1280, max: 1920 },
      height: { min: 576, ideal: 720, max: 1080 },
    },
  };

  navigator.mediaDevices
    .getUserMedia(constraints)
    .then(function (stream) {
      /* use the stream */
      let video = document.querySelector("#self-video");
      video.srcObject = stream;
      thisStream = stream;
      video.onloadedmetadata = function (e) {
        video.play();
      };
    })
    .catch(function (err) {
      /* handle the error */
      console.log(`Unable to use MediaStream API : ${err}`);
    });
</script>

<main>
  <video class="video" id="self-video" autoplay controls muted>
    <track kind="captions" />
  </video>

  <button class="states">{isHost ? "PEER" : "REMOTE PEER"}</button>
</main>

<style>
  main {
    width: 100%;
    height: 100%;
    margin: auto;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-evenly;
    background-color: #000;
    border-radius: 1rem;
  }
  .video {
    aspect-ratio: 16/9;
    background-color: transparent;
    width: 480px;
    border: 4px solid #3f3f3f;
    border-radius: 15px;
    cursor: pointer;
  }

  .states {
    position: absolute;
    bottom: 1rem;
    right: 1rem;
    padding: 0.5rem 1rem;
    text-transform: capitalize;
    border-radius: 0.5rem;
  }

  @media only screen and (min-width: 600px) {
    .video {
      aspect-ratio: 4/3;
      background-color: transparent;
      width: 240px;
    }
  }
</style>
