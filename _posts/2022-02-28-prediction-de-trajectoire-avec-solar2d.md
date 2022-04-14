---
category: Développement
tags: [Solar2D]
title: "Prédiction de trajectoire avec Solar2D"
---
Je crée en ce moment un jeu avec mon fils en m'appuyant sur [Solar 2D](https://solar2d.com/) ; un jeu pour
l'instant simple dans lequel il est possible de lancer un projectile en ajustant son coup au préalable,
à l'instar du lancé de pigeon dans un jeu comme Angry Birds. J'ai ajouté une prédiction de
trajectoire en partant du [tutorial existant](https://docs.coronalabs.com/tutorial/games/predictTrajectory/index.html)
dans la documentation officielle mais plusieurs points m'ont amené à revoir ma copie : un calcul de trajectoire à
adapter en fonction de la configuration ```display.fps``` ; une incompatibilité avec un lancement de projectile
utilisant ```applyLinearImpulse``` plutôt que ```setLinearVelocity``` ; une mise à jour de la prédiction qui suppose
que le projectile à lancer n'est pas en mouvement.

## Gestion des évènements
Pour mettre à jour la prédiction de trajectoire avec un projectile en mouvement, j'utilise deux évènements :
```touch``` pour déterminer ce que le joueur souhaite faire et ```lateUpdate``` pour réaliser la prédiction
de trajectoire.

```lua
-- ballImpulseForce est utilisé pour le calcul de la préduction lors de l'évènement lateUpdate
local ballImpulseForce = nil

local handleBallImpulseOnScreenTouch
local predictBallPathOnLateUpdate
local removePredictedBallPath

handleBallImpulseOnScreenTouch = function(event)
  -- Récupération de la distance et de la direction du lancé de projectile. Avec cette formule,
  -- le joueur tire en arrière pour lancer vers l'avant. Je multiplie par 2 pour augmenter la force
  local _ballImpulseForce = { x = (event.xStart - event.x) * 2, y = (event.yStart - event.y) * 2 }

  -- Par défaut rien n'est fait lors de l'évènement lateUpdate
  ballImpulseForce = nil

  if (event.phase == "ended") then
    -- ... Lancement du projectile
  elseif (event.phase == "moved") then
    -- La prédiction de trajectoire se fait uniquement lors de la phase moved de l'évènement touch
    ballImpulseForce = _ballImpulseForce
  end

  return true
end

predictBallPathOnLateUpdate = function()
  -- Prédiction de trajectoire uniquement lors de la phase moved de l'évènement touch
  if not ballImpulseForce then
    return false
  end
  -- ... Prédiction de la trajectoire
end

function scene:show(event)
  if event.phase == "did" then
    Runtime:addEventListener("touch", handleBallImpulseOnScreenTouch)
    Runtime:addEventListener("lateUpdate", predictBallPathOnLateUpdate)
  end
end

function scene:hide(event)
  if event.phase == "did" then
    Runtime:removeEventListener("lateUpdate", predictBallPathOnLateUpdate)
    Runtime:removeEventListener("touch", handleBallImpulseOnScreenTouch)
  end
end
```

## Prédiction de la trajectoire
J'utilise deux formules pour prédire la trajectoire du mon projectile ```scene.ball``` :

1. ```Acceleration = Force / Masse```
2. ```Distance parcourue du fait de la gravité = (Gravité * Temps²) / 2```

```lua
-- La valeur par défaut du ratio pixels par mètre utilisé par le moteur physique. Il est possible de
-- modifier cette valeur avec physics.setScale() mais pas de récupérer la valeur actuelle
local scale = 30

predictBallPathOnLateUpdate = function()
  if not ballImpulseForce then
    return false
  end

  -- Le temps en seconde entre les points qui seront affichés à l'écran
  local timeStepInterval = 0.1
  local gravityX, gravityY = physics.getGravity()

  -- La vélocité actuelle du projectile qui peut être en mouvement
  local velocityX, velocityY = scene.ball:getLinearVelocity()

  -- Suppression de la prédiction précédente
  removePredictedBallPath()

  -- Création du groupe auquel seront rattachés tous les points de la trajectoire
  scene.predictedBallPath = display.newGroup()
  scene.view:insert(scene.predictedBallPath)

  local prevStepX = nil
  local prevStepY = nil

  -- Je vais de 0 à 10 pour prédire une trajectoire sur un temps d'environ une seconde, cela peut être ajusté
  -- en modifiant timeStepInterval ou en jouant sur l'intervalle de la boucle for
  for step = 0, 10, 1 do
    local time = step * timeStepInterval

    -- Calcul de l'accélération
    local accelerationX = ballImpulseForce.x / scene.ball.mass
    local accelerationY = ballImpulseForce.y / scene.ball.mass

    -- Calcul des coordonnées du point de la trajectoire
    -- scene.ball.x : la position actuelle du projectile
    -- time * velocityX : la position future du projectile étant donné sa vitesse actuelle
    -- time * accelerationX : la position future du projectile étant donné son accelération
    -- 0.5 * gravityX * scale * (time * time) : la distance parcourue du fait de la gravité
    -- La gravité est multipliée par le ratio pixels par mètre car elle est exprimée en m/s
    local stepX = scene.ball.x + time * velocityX + time * accelerationX + 0.5 * gravityX * scale * (time * time)
    local stepY = scene.ball.y + time * velocityY + time * accelerationY + 0.5 * gravityY * scale * (time * time)

    -- Détection d'un éventuel obstacle entre ce point et le précédent
    if step > 0 and physics.rayCast(prevStepX, prevStepY, stepX, stepY, "any") then
      break
    end

    prevStepX = stepX
    prevStepY = stepY

    -- Ajout d'un point de prédiction de la trajectoire si aucun obstacle n'a été détecté
    display.newCircle(scene.predictedBallPath, stepX, stepY, 2)
  end

  return false
end

removePredictedBallPath = function()
  display.remove(scene.predictedBallPath)
  scene.predictedBallPath = nil
end
```

## Lancement du projectile
Lorsque le joueur relâche son doigt, le projectile est lancé.

```lua
handleBallImpulseOnScreenTouch = function(event)
  if (event.phase == "ended") then
    -- Suppression de la prédiction de trajectoire
    removePredictedBallPath()

    -- Impulsion sur le projectile en convertissant les pixels en mètres pour le moteur physique
    scene.ball:applyLinearImpulse(_ballImpulseForce.x / scale, _ballImpulseForce.y / scale, scene.ball.x, scene.ball.y)
  end
end
```

## Code complet
```lua
local composer = require "composer"
local physics = require "physics"

local ballImpulseForce = nil
local scale = 30
local scene = composer.newScene()

local handleBallImpulseOnScreenTouch
local predictBallPathOnLateUpdate
local removePredictedBallPath

handleBallImpulseOnScreenTouch = function(event)
  local _ballImpulseForce = { x = (event.xStart - event.x) * 2, y = (event.yStart - event.y) * 2 }
  ballImpulseForce = nil

  if (event.phase == "ended") then
    removePredictedBallPath()
    scene.ball:applyLinearImpulse(_ballImpulseForce.x / scale, _ballImpulseForce.y / scale, scene.ball.x, scene.ball.y)
  elseif (event.phase == "moved") then
    ballImpulseForce = _ballImpulseForce
  end

  return true
end

predictBallPathOnLateUpdate = function()
  if not ballImpulseForce then
    return false
  end

  local timeStepInterval = 0.1
  local gravityX, gravityY = physics.getGravity()
  local velocityX, velocityY = scene.ball:getLinearVelocity()

  removePredictedBallPath()
  scene.predictedBallPath = display.newGroup()
  scene.view:insert(scene.predictedBallPath)

  local prevStepX = nil
  local prevStepY = nil

  for step = 0, 10, 1 do
    local time = step * timeStepInterval
    local accelerationX = ballImpulseForce.x / scene.ball.mass
    local accelerationY = ballImpulseForce.y / scene.ball.mass
    local stepX = scene.ball.x + time * velocityX + time * accelerationX + 0.5 * gravityX * scale * (time * time)
    local stepY = scene.ball.y + time * velocityY + time * accelerationY + 0.5 * gravityY * scale * (time * time)

    if step > 0 and physics.rayCast(prevStepX, prevStepY, stepX, stepY, "any") then
      break
    end

    prevStepX = stepX
    prevStepY = stepY

    display.newCircle(scene.predictedBallPath, stepX, stepY, 2)
  end

  return false
end

removePredictedBallPath = function()
  display.remove(scene.predictedBallPath)
  scene.predictedBallPath = nil
end

function scene:create(event)
  physics.start()
  physics.pause()
  physics.setScale(scale);
  physics.setGravity(0, 9.8)

  -- ...

  self:createBall()
end

function scene:createBall()
  self.ball = display.newImageRect(self.view, "images/ball.png", 40, 40)
  self.ball.x = display.contentWidth / 2
  self.ball.y = display.contentHeight / 2

  physics.addBody(self.ball, {
    radius = self.ball.width / 2 - 1,
    density = 1.0,
    friction = 0.3,
    bounce = 0.5,
  })

  self.ball.angularDamping = 1.5
end

function scene:show(event)
  if event.phase == "did" then
    Runtime:addEventListener("touch", handleBallImpulseOnScreenTouch)
    Runtime:addEventListener("lateUpdate", predictBallPathOnLateUpdate)
    physics.start()
  end
end

function scene:hide(event)
  if event.phase == "did" then
    Runtime:removeEventListener("lateUpdate", predictBallPathOnLateUpdate)
    Runtime:removeEventListener("touch", handleBallImpulseOnScreenTouch)
    physics.stop()

    composer.removeScene("game")
  end
end

function scene:destroy(event)
  package.loaded[physics] = nil
  physics = nil
end

scene:addEventListener("create", scene)
scene:addEventListener("show", scene)
scene:addEventListener("hide", scene)
scene:addEventListener("destroy", scene)

return scene
```
