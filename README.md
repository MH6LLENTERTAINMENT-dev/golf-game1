import { useRef, useState, useEffect } from "react";
import { useFrame, useThree } from "@react-three/fiber";
import { Sphere, Plane, OrbitControls } from "@react-three/drei";

// Define 18 holes (simple layout for demo)
const HOLES = [
  { hole: 1, par: 4, position: [0, 0, -15] },
  { hole: 2, par: 3, position: [10, 0, -20] },
  { hole: 3, par: 5, position: [-12, 0, -18] },
  { hole: 4, par: 4, position: [15, 0, -25] },
  { hole: 5, par: 3, position: [0, 0, -30] },
  { hole: 6, par: 4, position: [-15, 0, -22] },
  { hole: 7, par: 5, position: [20, 0, -28] },
  { hole: 8, par: 3, position: [-10, 0, -35] },
  { hole: 9, par: 4, position: [12, 0, -32] },
  { hole: 10, par: 4, position: [5, 0, -40] },
  { hole: 11, par: 3, position: [-20, 0, -38] },
  { hole: 12, par: 5, position: [18, 0, -42] },
  { hole: 13, par: 4, position: [-5, 0, -45] },
  { hole: 14, par: 4, position: [15, 0, -48] },
  { hole: 15, par: 5, position: [0, 0, -50] },
  { hole: 16, par: 3, position: [-15, 0, -55] },
  { hole: 17, par: 4, position: [10, 0, -58] },
  { hole: 18, par: 5, position: [0, 0, -60] }
];

export default function MiniGolf() {
  const ballRef = useRef();
  const { camera } = useThree();

  // Game state
  const [currentHole, setCurrentHole] = useState(0);
  const [ballPos, setBallPos] = useState([0, 0.5, 0]);
  const [velocity, setVelocity] = useState([0, 0, 0]);
  const [strokes, setStrokes] = useState(0);
  const [scores, setScores] = useState(Array(18).fill(0));

  // Shot state
  const [power, setPower] = useState(0.2);
  const [direction, setDirection] = useState([0, 0, -1]);

  // Background music
  useEffect(() => {
    const audio = new Audio("https://actions.google.com/sounds/v1/ambiences/forest_day.ogg");
    audio.loop = true;
    audio.volume = 0.4;
    audio.play().catch(() => console.log("Music starts after first user action"));
    return () => audio.pause();
  }, []);

  // Handle controls
  useEffect(() => {
    const handleKey = (e) => {
      if (e.code === "Space") {
        // Hit ball
        setVelocity([direction[0] * power, 0, direction[2] * power]);
        setStrokes((s) => s + 1);
      }
      if (e.code === "KeyW" || e.code === "ArrowUp") setDirection([0, 0, -1]);
      if (e.code === "KeyS" || e.code === "ArrowDown") setDirection([0, 0, 1]);
      if (e.code === "KeyA" || e.code === "ArrowLeft") setDirection([-1, 0, 0]);
      if (e.code === "KeyD" || e.code === "ArrowRight") setDirection([1, 0, 0]);
      if (e.code === "KeyQ") setPower((p) => Math.min(p + 0.1, 1.5));
      if (e.code === "KeyE") setPower((p) => Math.max(p - 0.1, 0.1));
      if (e.code === "KeyZ") camera.position.z -= 2;
      if (e.code === "KeyX") camera.position.z += 2;
    };
    window.addEventListener("keydown", handleKey);
    return () => window.removeEventListener("keydown", handleKey);
  }, [direction, power, camera]);

  // Ball movement + hole detection
  useFrame(() => {
    if (!ballRef.current) return;

    let [x, y, z] = ballPos;
    const [vx, vy, vz] = velocity;

    // Apply velocity
    x += vx;
    z += vz;

    // Apply friction
    const friction = 0.97;
    const newVx = vx * friction;
    const newVz = vz * friction;

    if (Math.abs(newVx) < 0.001 && Math.abs(newVz) < 0.001) {
      setVelocity([0, 0, 0]);
    } else {
      setVelocity([newVx, vy, newVz]);
    }

    setBallPos([x, y, z]);
    ballRef.current.position.set(x, y, z);

    // Hole detection
    const hole = HOLES[currentHole];
    const dx = x - hole.position[0];
    const dz = z - hole.position[2];
    const dist = Math.sqrt(dx * dx + dz * dz);

    if (dist < 1.5) {
      // Ball in hole
      const newScores = [...scores];
      newScores[currentHole] = strokes;
      setScores(newScores);

      // Next hole or finish
      if (currentHole < HOLES.length - 1) {
        setCurrentHole((h) => h + 1);
        setBallPos([0, 0.5, 0]);
        setVelocity([0, 0, 0]);
        setStrokes(0);
      } else {
        alert(`Game Over! Total Strokes: ${newScores.reduce((a, b) => a + b, 0)}`);
      }
    }
  });

  return (
    <>
      {/* Course ground */}
      <Plane args={[80, 100]} rotation={[-Math.PI / 2, 0, 0]} receiveShadow>
        <meshStandardMaterial color="green" />
      </Plane>

      {/* Ball */}
      <Sphere ref={ballRef} args={[0.5, 32, 32]} castShadow>
        <meshStandardMaterial color="white" />
      </Sphere>

      {/* Current hole */}
      <mesh position={HOLES[currentHole].position}>
        <cylinderGeometry args={[1, 1, 0.2, 32]} />
        <meshStandardMaterial color="black" />
      </mesh>

      {/* Arrow shows direction */}
      <mesh position={[ballPos[0] + direction[0] * 2, 0.5, ballPos[2] + direction[2] * 2]}>
        <coneGeometry args={[0.3, 1, 16]} />
        <meshStandardMaterial color="red" />
      </mesh>

      {/* Lighting */}
      <ambientLight intensity={0.6} />
      <directionalLight position={[10, 20, 10]} intensity={1} />

      {/* Camera controls */}
      <OrbitControls />

      {/* HUD (text overlay) */}
      <Html position={[0, 5, 0]}>
        <div style={{ 
          background: "rgba(0,0,0,0.6)", 
          color: "white", 
          padding: "6px 10px", 
          borderRadius: "6px" 
        }}>
          <div>Hole {HOLES[currentHole].hole} (Par {HOLES[currentHole].par})</div>
          <div>Strokes: {strokes}</div>
          <div>Power: {power.toFixed(1)}</div>
        </div>
      </Html>
    </>
  );
}
